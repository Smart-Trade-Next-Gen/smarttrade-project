# Design Document: smarttrade_common.instrument_master

**Title**: Instrument Master Reference Data Library  
**Status**: Design (Ready for Implementation)  
**Version**: 1.0  
**Date**: 2026-04-24  

---

## Executive Summary

This document defines the design of the `smarttrade_common.instrument_master` package — a shared library that provides instrument metadata replication, validation, and lookup across all SmartTrade services.

**Goals**:
1. Eliminate runtime MDS dependency for symbol/instrument resolution
2. Provide unified InstrumentRegistry and validation APIs to all services
3. Implement snapshot replication pattern (bootstrap + periodic refresh)
4. Reduce code duplication (currently each service has its own symbol logic)
5. Support deterministic order execution with <1ms instrument lookups

**Scope**: 
- Canonical Instrument data model
- In-memory InstrumentRegistry for fast lookups
- Persistent InstrumentCache for recovery across restarts
- InstrumentSyncService for MDS synchronization
- Shared validation APIs (tick, lot, trading hours)

**Out of Scope (Phase 2+)**:
- Event-driven incremental updates
- Multi-exchange instrument arbitrage
- Instrument alias mapping
- Instrument-specific trading rules engine

---

## 1. Data Model & Schema

### 1.1 Canonical Instrument Class

```python
# smarttrade_common/instrument_master/models.py

from dataclasses import dataclass, field
from datetime import datetime, date
from enum import Enum
from typing import Optional
from uuid import UUID

class InstrumentType(str, Enum):
    """Instrument classification."""
    EQUITY = "EQUITY"
    FUTURE = "FUTURE"
    OPTION = "OPTION"
    CURRENCY = "CURRENCY"
    COMMODITY = "COMMODITY"
    INDEX = "INDEX"
    ETF = "ETF"
    FUND = "FUND"
    BOND = "BOND"

class InstrumentStatus(str, Enum):
    """Instrument trading status."""
    ACTIVE = "ACTIVE"
    SUSPENDED = "SUSPENDED"
    DELISTED = "DELISTED"
    INACTIVE = "INACTIVE"

class OptionType(str, Enum):
    """Option classification."""
    CALL = "CALL"
    PUT = "PUT"

@dataclass
class Instrument:
    """
    Canonical instrument master record.
    
    Immutable dataclass representing a single trading instrument.
    Replicated across services; source of truth in MDS.
    """
    
    # === IDENTITY ===
    instrument_id: UUID
    """Unique identifier; never changes."""
    
    symbol: str
    """Trading symbol (e.g., 'NSE:INFY', 'MCX:GOLD', 'NCDEX:COTTON').
    
    Format: EXCHANGE:SYMBOL
    - NSE: National Stock Exchange (Equities)
    - BSE: Bombay Stock Exchange (Equities)
    - NCDEX: Agricultural commodities
    - MCX: Precious metals, energy
    - SEBI: SENSEX index
    - NIFTY: NIFTY index
    """
    
    exchange: str
    """Exchange code (NSE, BSE, MCX, NCDEX, etc.)."""
    
    isin: Optional[str] = None
    """ISIN code (if applicable; mostly for equities)."""
    
    # === CONTRACT DETAILS ===
    instrument_type: InstrumentType
    """Classification of instrument."""
    
    expiry_date: Optional[date] = None
    """Expiration date (for futures, options). None for perpetuals."""
    
    strike_price: Optional[float] = None
    """Strike price (for options only)."""
    
    option_type: Optional[OptionType] = None
    """Call or Put (for options only)."""
    
    underlying_symbol: Optional[str] = None
    """Symbol of underlying instrument (for derivatives)."""
    
    # === VALIDATION RULES ===
    tick_size: float
    """Minimum price increment.
    
    Example: 0.05 means prices must be multiples of 0.05.
    Used to validate order price; rejected if price % tick_size != 0.
    """
    
    lot_size: int
    """Standard lot size; minimum tradeable quantity.
    
    Example: 1 for equities, 100 for some futures.
    Used to validate order quantity; rejected if quantity % lot_size != 0.
    """
    
    min_price: Optional[float] = None
    """Minimum permissible order price (if exchange enforces)."""
    
    max_price: Optional[float] = None
    """Maximum permissible order price (if exchange enforces)."""
    
    multiplier: float = 1.0
    """Contract multiplier (for futures/options).
    
    Example: For Nifty50 futures, multiplier=50 (1 lot = 50 * index value).
    """
    
    # === TRADING DETAILS ===
    sector: Optional[str] = None
    """Industry sector (for equities). E.g., 'IT', 'PHARMA', 'FINANCE'."""
    
    industry: Optional[str] = None
    """Industry classification (for equities)."""
    
    market_cap_category: Optional[str] = None
    """'LARGE_CAP', 'MID_CAP', 'SMALL_CAP' (for equities)."""
    
    status: InstrumentStatus = InstrumentStatus.ACTIVE
    """Current trading status."""
    
    description: Optional[str] = None
    """Human-readable description."""
    
    # === VERSIONING & INTEGRITY ===
    version: int
    """Version number; incremented on any metadata change.
    
    Used to detect stale replicas and for optimistic locking.
    """
    
    checksum: str
    """SHA256 hash of canonical form.
    
    Used for integrity verification across replicas.
    Computed as: SHA256(symbol + exchange + tick_size + lot_size + ... )
    """
    
    # === TIMESTAMPS ===
    created_at: datetime
    """When this instrument was first added to master."""
    
    updated_at: datetime
    """When this instrument was last updated."""
    
    # === INTERNAL STATE ===
    _cached_repr: str = field(default="", repr=False)
    """Internal cache for string representation."""

    def __post_init__(self):
        """Validate invariants."""
        if not self.symbol or ':' not in self.symbol:
            raise ValueError(f"Invalid symbol format: {self.symbol}. Must be EXCHANGE:SYMBOL")
        
        if self.tick_size <= 0:
            raise ValueError(f"tick_size must be positive: {self.tick_size}")
        
        if self.lot_size <= 0:
            raise ValueError(f"lot_size must be positive: {self.lot_size}")
        
        if self.instrument_type == InstrumentType.OPTION:
            if not self.strike_price or not self.option_type:
                raise ValueError("Options require strike_price and option_type")
        
        if self.instrument_type in [InstrumentType.FUTURE, InstrumentType.OPTION]:
            if not self.expiry_date:
                raise ValueError(f"{self.instrument_type} requires expiry_date")
    
    def __hash__(self):
        """Hash by symbol (immutable identity)."""
        return hash(self.symbol)
    
    def __eq__(self, other):
        """Equality by symbol."""
        if not isinstance(other, Instrument):
            return False
        return self.symbol == other.symbol
    
    def validate_order_price(self, price: float) -> bool:
        """Check if price respects tick_size rules."""
        if self.tick_size == 0:
            return True
        
        # Check: (price % tick_size) == 0 within floating-point tolerance
        remainder = price % self.tick_size
        tolerance = 1e-9
        return abs(remainder) < tolerance or abs(remainder - self.tick_size) < tolerance
    
    def validate_order_quantity(self, quantity: int) -> bool:
        """Check if quantity is valid multiple of lot_size."""
        return quantity > 0 and (quantity % self.lot_size) == 0
    
    def is_tradeable(self) -> bool:
        """Can this instrument be traded now?"""
        return self.status == InstrumentStatus.ACTIVE
    
    def is_derivative(self) -> bool:
        """Is this a derivative (futures, options)?"""
        return self.instrument_type in [
            InstrumentType.FUTURE,
            InstrumentType.OPTION,
            InstrumentType.CURRENCY
        ]
    
    def to_dict(self) -> dict:
        """Serialize to dict for storage/transmission."""
        return {
            'instrument_id': str(self.instrument_id),
            'symbol': self.symbol,
            'exchange': self.exchange,
            'isin': self.isin,
            'instrument_type': self.instrument_type.value,
            'expiry_date': self.expiry_date.isoformat() if self.expiry_date else None,
            'strike_price': self.strike_price,
            'option_type': self.option_type.value if self.option_type else None,
            'underlying_symbol': self.underlying_symbol,
            'tick_size': self.tick_size,
            'lot_size': self.lot_size,
            'min_price': self.min_price,
            'max_price': self.max_price,
            'multiplier': self.multiplier,
            'sector': self.sector,
            'industry': self.industry,
            'status': self.status.value,
            'version': self.version,
            'checksum': self.checksum,
            'created_at': self.created_at.isoformat(),
            'updated_at': self.updated_at.isoformat(),
        }
    
    @classmethod
    def from_dict(cls, data: dict) -> 'Instrument':
        """Deserialize from dict."""
        from datetime import datetime as dt
        
        return cls(
            instrument_id=UUID(data['instrument_id']),
            symbol=data['symbol'],
            exchange=data['exchange'],
            isin=data.get('isin'),
            instrument_type=InstrumentType(data['instrument_type']),
            expiry_date=dt.fromisoformat(data['expiry_date']).date()
                if data.get('expiry_date') else None,
            strike_price=data.get('strike_price'),
            option_type=OptionType(data['option_type']) if data.get('option_type') else None,
            underlying_symbol=data.get('underlying_symbol'),
            tick_size=data['tick_size'],
            lot_size=data['lot_size'],
            min_price=data.get('min_price'),
            max_price=data.get('max_price'),
            multiplier=data.get('multiplier', 1.0),
            sector=data.get('sector'),
            industry=data.get('industry'),
            status=InstrumentStatus(data.get('status', 'ACTIVE')),
            version=data['version'],
            checksum=data['checksum'],
            created_at=dt.fromisoformat(data['created_at']),
            updated_at=dt.fromisoformat(data['updated_at']),
        )
```

### 1.2 Validation Result

```python
@dataclass
class ValidationResult:
    """Result of instrument/order validation."""
    
    is_valid: bool
    """Whether validation passed."""
    
    errors: List[str] = field(default_factory=list)
    """List of validation errors (if is_valid=False)."""
    
    instrument: Optional[Instrument] = None
    """Reference to instrument (if found)."""
    
    details: dict = field(default_factory=dict)
    """Additional context (e.g., tick_size, lot_size)."""
    
    def add_error(self, message: str) -> 'ValidationResult':
        """Add an error and mark as invalid."""
        self.is_valid = False
        self.errors.append(message)
        return self
    
    def __bool__(self):
        """Truthy if valid."""
        return self.is_valid
```

---

## 2. InstrumentRegistry: In-Memory Lookup

### 2.1 Core Class

```python
# smarttrade_common/instrument_master/cache.py

from typing import Dict, List, Optional
from collections import defaultdict
import threading

class InstrumentRegistry:
    """
    Fast, thread-safe, in-memory lookup of instruments.
    
    Provides O(1) lookups by symbol and filtered searches.
    Immutable snapshots; updates via replace_all().
    
    **Usage**:
        registry = InstrumentRegistry.from_instruments([...])
        instrument = registry.get("NSE:INFY")
        instruments = registry.search(exchange="NSE", instrument_type="EQUITY")
    """
    
    def __init__(self, instruments: Dict[str, Instrument]):
        """
        Initialize with instruments dict.
        
        Args:
            instruments: Dict[symbol -> Instrument]
        """
        self._instruments = instruments
        self._lock = threading.RLock()
        
        # Secondary indices for fast searches
        self._by_exchange: Dict[str, List[Instrument]] = defaultdict(list)
        self._by_type: Dict[str, List[Instrument]] = defaultdict(list)
        self._by_status: Dict[str, List[Instrument]] = defaultdict(list)
        
        self._build_indices()
    
    def _build_indices(self) -> None:
        """Rebuild secondary indices."""
        self._by_exchange.clear()
        self._by_type.clear()
        self._by_status.clear()
        
        for instr in self._instruments.values():
            self._by_exchange[instr.exchange].append(instr)
            self._by_type[instr.instrument_type.value].append(instr)
            self._by_status[instr.status.value].append(instr)
    
    def get(self, symbol: str) -> Optional[Instrument]:
        """
        Get instrument by symbol.
        
        Args:
            symbol: e.g., 'NSE:INFY'
        
        Returns:
            Instrument or None if not found.
        
        Time Complexity: O(1)
        """
        with self._lock:
            return self._instruments.get(symbol)
    
    def search(
        self,
        exchange: Optional[str] = None,
        instrument_type: Optional[str] = None,
        status: Optional[str] = None,
    ) -> List[Instrument]:
        """
        Search instruments by criteria.
        
        Args:
            exchange: Filter by exchange (e.g., 'NSE')
            instrument_type: Filter by type (e.g., 'EQUITY')
            status: Filter by status (e.g., 'ACTIVE')
        
        Returns:
            List of matching instruments.
        
        Example:
            registry.search(exchange='NSE', instrument_type='EQUITY')
        """
        with self._lock:
            results = set(self._instruments.values())
            
            if exchange:
                results &= set(self._by_exchange.get(exchange, []))
            
            if instrument_type:
                results &= set(self._by_type.get(instrument_type, []))
            
            if status:
                results &= set(self._by_status.get(status, []))
            
            return sorted(list(results), key=lambda x: x.symbol)
    
    def validate_order_price(self, symbol: str, price: float) -> ValidationResult:
        """Validate order price against tick_size."""
        result = ValidationResult(is_valid=False)
        
        instrument = self.get(symbol)
        if not instrument:
            return result.add_error(f"Instrument {symbol} not found")
        
        result.instrument = instrument
        
        if not instrument.validate_order_price(price):
            return result.add_error(
                f"Price {price} violates tick_size {instrument.tick_size}"
            )
        
        result.is_valid = True
        return result
    
    def validate_order_quantity(self, symbol: str, quantity: int) -> ValidationResult:
        """Validate order quantity against lot_size."""
        result = ValidationResult(is_valid=False)
        
        instrument = self.get(symbol)
        if not instrument:
            return result.add_error(f"Instrument {symbol} not found")
        
        result.instrument = instrument
        
        if quantity <= 0:
            return result.add_error(f"Quantity must be positive: {quantity}")
        
        if not instrument.validate_order_quantity(quantity):
            return result.add_error(
                f"Quantity {quantity} not multiple of lot_size {instrument.lot_size}"
            )
        
        result.is_valid = True
        return result
    
    def validate_order(self, symbol: str, quantity: int, price: float) -> ValidationResult:
        """Comprehensive order validation."""
        result = ValidationResult(is_valid=True)
        
        instrument = self.get(symbol)
        if not instrument:
            return result.add_error(f"Instrument {symbol} not found")
        
        result.instrument = instrument
        
        # Check tradeable
        if not instrument.is_tradeable():
            result.add_error(f"Instrument {symbol} is not tradeable ({instrument.status.value})")
        
        # Check quantity
        if not instrument.validate_order_quantity(quantity):
            result.add_error(
                f"Quantity {quantity} not multiple of lot_size {instrument.lot_size}"
            )
        
        # Check price
        if not instrument.validate_order_price(price):
            result.add_error(
                f"Price {price} violates tick_size {instrument.tick_size}"
            )
        
        # Check price bounds
        if instrument.min_price and price < instrument.min_price:
            result.add_error(f"Price {price} below minimum {instrument.min_price}")
        if instrument.max_price and price > instrument.max_price:
            result.add_error(f"Price {price} above maximum {instrument.max_price}")
        
        return result
    
    def snapshot(self) -> Dict[str, Instrument]:
        """Get immutable snapshot of all instruments."""
        with self._lock:
            return dict(self._instruments)
    
    def size(self) -> int:
        """Number of instruments."""
        return len(self._instruments)
    
    def is_empty(self) -> bool:
        """Are there any instruments?"""
        return len(self._instruments) == 0
    
    @classmethod
    def from_instruments(cls, instruments: List[Instrument]) -> 'InstrumentRegistry':
        """Construct from list."""
        return cls({instr.symbol: instr for instr in instruments})
```

---

## 3. InstrumentCache: Persistent Storage

### 3.1 Cache Interface

```python
# smarttrade_common/instrument_master/repository.py

from abc import ABC, abstractmethod
from typing import List, Optional
from datetime import datetime
import hashlib
import json

class InstrumentCacheBackend(ABC):
    """Abstract backend for instrument storage."""
    
    @abstractmethod
    async def load_all(self) -> List[Instrument]:
        """Load all instruments from storage."""
    
    @abstractmethod
    async def save_all(self, instruments: List[Instrument]) -> None:
        """Persist all instruments."""
    
    @abstractmethod
    async def get_metadata(self) -> dict:
        """Get cache metadata (version, checksum, sync_time)."""
    
    @abstractmethod
    async def set_metadata(self, metadata: dict) -> None:
        """Set cache metadata."""

class InstrumentCache:
    """
    Persistent cache for instrument master.
    
    Survives service restart; used to bootstrap InstrumentRegistry.
    Handles versioning and checksum validation.
    
    **Backends**:
    - Database (PostgreSQL, SQLite)
    - Redis (for cache layer)
    - File-based (dev/testing)
    """
    
    def __init__(self, backend: InstrumentCacheBackend):
        """
        Args:
            backend: Storage backend (DB, Redis, etc.)
        """
        self.backend = backend
        self._registry: Optional[InstrumentRegistry] = None
        self._version: int = 0
        self._checksum: str = ""
        self._last_sync_time: Optional[datetime] = None
    
    async def load(self) -> InstrumentRegistry:
        """
        Load instruments from storage and return registry.
        
        Returns:
            InstrumentRegistry (may be empty if cache miss).
        
        Raises:
            CacheError: If storage backend fails.
        """
        try:
            instruments = await self.backend.load_all()
            metadata = await self.backend.get_metadata()
            
            self._version = metadata.get('version', 0)
            self._checksum = metadata.get('checksum', '')
            self._last_sync_time = metadata.get('last_sync_time')
            
            self._registry = InstrumentRegistry.from_instruments(instruments)
            return self._registry
        
        except Exception as e:
            # Graceful degradation: return empty registry
            logger.error(f"Failed to load instrument cache: {e}")
            self._registry = InstrumentRegistry.from_instruments([])
            return self._registry
    
    async def upsert(self, instruments: List[Instrument]) -> bool:
        """
        Merge new/updated instruments; idempotent.
        
        Compares versions and checksums; only persists if changed.
        
        Args:
            instruments: New/updated instruments from MDS.
        
        Returns:
            True if cache was updated; False if unchanged.
        """
        new_checksum = self._compute_checksum(instruments)
        
        if new_checksum == self._checksum:
            logger.debug("Instrument cache unchanged (checksum match)")
            return False
        
        # Merge: keep existing, add/update from new list
        merged = dict(self._registry.snapshot()) if self._registry else {}
        
        for instr in instruments:
            if instr.symbol in merged:
                # Update existing: only if version is newer
                if instr.version > merged[instr.symbol].version:
                    merged[instr.symbol] = instr
            else:
                # Add new
                merged[instr.symbol] = instr
        
        # Persist
        await self.backend.save_all(list(merged.values()))
        
        # Update metadata
        self._version += 1
        self._checksum = new_checksum
        self._last_sync_time = datetime.utcnow()
        
        metadata = {
            'version': self._version,
            'checksum': self._checksum,
            'last_sync_time': self._last_sync_time.isoformat(),
        }
        await self.backend.set_metadata(metadata)
        
        # Refresh registry
        self._registry = InstrumentRegistry.from_instruments(list(merged.values()))
        
        logger.info(f"Instrument cache updated: {len(merged)} instruments")
        return True
    
    async def clear(self) -> None:
        """
        Full reset of cache (dangerous; used for recovery).
        """
        await self.backend.save_all([])
        self._version = 0
        self._checksum = ""
        self._last_sync_time = None
        self._registry = InstrumentRegistry.from_instruments([])
        logger.warning("Instrument cache cleared")
    
    @property
    def version(self) -> int:
        """Current cache version."""
        return self._version
    
    @property
    def checksum(self) -> str:
        """Current cache checksum."""
        return self._checksum
    
    @property
    def last_sync_time(self) -> Optional[datetime]:
        """When was cache last synced from MDS?"""
        return self._last_sync_time
    
    @property
    def is_stale(self, max_age_seconds: int = 21600) -> bool:
        """Is cache older than max_age?"""
        if not self._last_sync_time:
            return True
        
        age = (datetime.utcnow() - self._last_sync_time).total_seconds()
        return age > max_age_seconds  # 6 hours default
    
    def get_registry(self) -> InstrumentRegistry:
        """Get current in-memory registry."""
        if self._registry is None:
            self._registry = InstrumentRegistry.from_instruments([])
        return self._registry
    
    @staticmethod
    def _compute_checksum(instruments: List[Instrument]) -> str:
        """Compute SHA256 of canonical instrument list."""
        # Canonical order: by symbol
        sorted_instrs = sorted(instruments, key=lambda x: x.symbol)
        
        # Serialize to JSON
        data = json.dumps(
            [instr.to_dict() for instr in sorted_instrs],
            sort_keys=True,
            default=str
        )
        
        return hashlib.sha256(data.encode()).hexdigest()
```

---

## 4. InstrumentSyncService: Bootstrap & Refresh

### 4.1 Sync Service

```python
# smarttrade_common/instrument_master/sync_service.py

from datetime import datetime, timedelta
from typing import List, Optional
from enum import Enum
import logging

logger = logging.getLogger(__name__)

class SyncStatus(str, Enum):
    """Synchronization status."""
    PENDING = "PENDING"
    IN_PROGRESS = "IN_PROGRESS"
    SUCCESS = "SUCCESS"
    FAILED = "FAILED"

class InstrumentSyncService:
    """
    Orchestrates instrument master synchronization from MDS.
    
    Handles:
    - One-time bootstrap (startup)
    - Periodic refresh (e.g., every 6 hours)
    - Failure recovery (retries, fallbacks)
    - Staleness detection
    
    **Usage**:
        service = InstrumentSyncService(cache, mds_client)
        await service.bootstrap()  # One-time at startup
        # Later, in background job:
        await service.refresh()
    """
    
    def __init__(
        self,
        cache: InstrumentCache,
        mds_client: 'BaseServiceClient',
        max_retries: int = 3,
        retry_backoff_seconds: int = 5,
    ):
        """
        Args:
            cache: InstrumentCache for persistence.
            mds_client: Client for calling MDS APIs.
            max_retries: Max retry attempts on failure.
            retry_backoff_seconds: Exponential backoff base.
        """
        self.cache = cache
        self.mds_client = mds_client
        self.max_retries = max_retries
        self.retry_backoff_seconds = retry_backoff_seconds
        
        self._last_status = SyncStatus.PENDING
        self._last_error: Optional[Exception] = None
        self._failure_count: int = 0
    
    async def bootstrap(self) -> InstrumentRegistry:
        """
        One-time bootstrap at service startup.
        
        1. Try to load from persistent cache
        2. If cache is empty/fresh:
           - Fetch full instrument list from MDS
           - Persist to cache
           - Return registry
        3. If MDS unavailable:
           - Return empty registry (service starts; orders rejected)
           - Alert ops to investigate
        
        Returns:
            InstrumentRegistry (may be empty if both cache and MDS fail).
        
        Raises:
            BootstrapError: If unable to start (critical condition).
        """
        logger.info("Bootstrapping instrument master...")
        
        # Step 1: Try to load from cache
        registry = await self.cache.load()
        
        if not registry.is_empty() and not self.cache.is_stale(max_age_seconds=86400):
            logger.info(f"Loaded {registry.size()} instruments from cache (fresh)")
            return registry
        
        # Step 2: Cache is empty or stale; fetch from MDS
        logger.info("Fetching instruments from MDS...")
        
        for attempt in range(1, self.max_retries + 1):
            try:
                instruments = await self._fetch_from_mds()
                
                if not instruments:
                    raise Exception("MDS returned empty instrument list")
                
                # Persist
                await self.cache.upsert(instruments)
                
                registry = self.cache.get_registry()
                logger.info(f"Bootstrapped {registry.size()} instruments from MDS")
                
                self._last_status = SyncStatus.SUCCESS
                self._failure_count = 0
                return registry
            
            except Exception as e:
                logger.warning(f"Bootstrap attempt {attempt}/{self.max_retries} failed: {e}")
                
                if attempt < self.max_retries:
                    wait_seconds = self.retry_backoff_seconds ** attempt
                    await asyncio.sleep(wait_seconds)
        
        # All retries exhausted
        self._last_status = SyncStatus.FAILED
        self._last_error = e
        self._failure_count += 1
        
        logger.error(f"Bootstrap failed after {self.max_retries} attempts")
        
        # Return empty registry; service will reject all orders
        return self.cache.get_registry()
    
    async def refresh(self, force: bool = False) -> bool:
        """
        Periodic refresh of instrument master.
        
        Called every 6 hours (or manually via force=True).
        Idempotent: no-op if cache is fresh (checksum unchanged).
        
        Args:
            force: Force refresh even if cache appears fresh.
        
        Returns:
            True if cache was updated; False if unchanged.
        """
        logger.debug("Refreshing instrument master...")
        
        if self._last_status == SyncStatus.IN_PROGRESS:
            logger.debug("Refresh already in progress; skipping")
            return False
        
        self._last_status = SyncStatus.IN_PROGRESS
        
        try:
            instruments = await self._fetch_from_mds()
            
            if not instruments:
                logger.warning("MDS returned empty list; skipping refresh")
                return False
            
            # Merge into cache (idempotent)
            updated = await self.cache.upsert(instruments)
            
            if updated:
                logger.info(f"Instrument cache updated")
                self._failure_count = 0
            else:
                logger.debug("Instrument cache unchanged")
            
            self._last_status = SyncStatus.SUCCESS
            return updated
        
        except Exception as e:
            logger.error(f"Refresh failed: {e}")
            self._last_status = SyncStatus.FAILED
            self._last_error = e
            self._failure_count += 1
            
            # Alert if too many consecutive failures
            if self._failure_count >= 3:
                logger.critical(
                    f"Instrument sync failed {self._failure_count} times; "
                    "cache may be stale. Alert ops."
                )
            
            return False
    
    async def _fetch_from_mds(self) -> List[Instrument]:
        """
        Fetch full instrument list from MDS API.
        
        Calls: GET /api/v1/instruments
        
        Returns:
            List of Instrument objects.
        
        Raises:
            Exception: If MDS request fails.
        """
        response = await self.mds_client.request(
            method="GET",
            path="/api/v1/instruments",
            timeout=30  # seconds
        )
        
        if not response.ok:
            raise Exception(f"MDS request failed: {response.status_code}")
        
        data = response.json()
        instruments = [Instrument.from_dict(item) for item in data.get('instruments', [])]
        
        return instruments
    
    @property
    def last_status(self) -> SyncStatus:
        """Last sync attempt status."""
        return self._last_status
    
    @property
    def last_error(self) -> Optional[Exception]:
        """Last sync error (if any)."""
        return self._last_error
    
    @property
    def failure_count(self) -> int:
        """Number of consecutive failures."""
        return self._failure_count
    
    def is_healthy(self) -> bool:
        """Is sync service in good state?"""
        return (
            self._last_status == SyncStatus.SUCCESS
            and not self.cache.is_stale()
            and self._failure_count == 0
        )
    
    def sync_lag(self) -> Optional[timedelta]:
        """How old is the cache compared to now?"""
        if not self.cache.last_sync_time:
            return None
        
        return datetime.utcnow() - self.cache.last_sync_time
```

---

## 5. Validators: Shared Validation APIs

```python
# smarttrade_common/instrument_master/validators.py

class InstrumentValidator:
    """
    Validation helper for order placement and strategy evaluation.
    
    Provides static methods for instrument/order validation.
    Used by BAS, PBS, Strategy, and any service that needs validation.
    """
    
    @staticmethod
    def validate_symbol(symbol: str, registry: InstrumentRegistry) -> ValidationResult:
        """
        Validate that symbol exists and is tradeable.
        
        Args:
            symbol: Trading symbol (e.g., 'NSE:INFY').
            registry: InstrumentRegistry for lookup.
        
        Returns:
            ValidationResult with details.
        """
        result = ValidationResult(is_valid=False)
        
        instrument = registry.get(symbol)
        if not instrument:
            return result.add_error(f"Symbol {symbol} not found")
        
        if not instrument.is_tradeable():
            return result.add_error(f"Symbol {symbol} is not tradeable ({instrument.status.value})")
        
        result.is_valid = True
        result.instrument = instrument
        return result
    
    @staticmethod
    def validate_order(
        symbol: str,
        quantity: int,
        price: float,
        registry: InstrumentRegistry,
    ) -> ValidationResult:
        """
        Comprehensive order validation.
        
        Checks:
        - Symbol exists and is tradeable
        - Quantity is positive and multiple of lot_size
        - Price is positive and multiple of tick_size
        - Price is within min/max bounds (if set)
        
        Args:
            symbol: Trading symbol.
            quantity: Order quantity.
            price: Order price.
            registry: InstrumentRegistry.
        
        Returns:
            ValidationResult with detailed errors.
        
        Example:
            result = InstrumentValidator.validate_order(
                'NSE:INFY', 1, 1500.0, registry
            )
            if not result:
                raise OrderError(result.errors)
        """
        return registry.validate_order(symbol, quantity, price)
    
    @staticmethod
    def validate_trading_hours(
        symbol: str,
        timestamp: datetime,
        registry: InstrumentRegistry,
        calendar: 'TradingCalendar',
    ) -> ValidationResult:
        """
        Check if symbol can be traded at given time.
        
        Checks:
        - Exchange is open
        - Symbol is not suspended during this time
        - No auction/circuit breaker period
        
        Args:
            symbol: Trading symbol.
            timestamp: Time to check.
            registry: InstrumentRegistry.
            calendar: TradingCalendar for market hours.
        
        Returns:
            ValidationResult.
        """
        result = ValidationResult(is_valid=True)
        
        instrument = registry.get(symbol)
        if not instrument:
            return result.add_error(f"Symbol {symbol} not found")
        
        if not calendar.is_market_open(instrument.exchange, timestamp):
            return result.add_error(
                f"Market {instrument.exchange} is not open at {timestamp}"
            )
        
        result.is_valid = True
        result.instrument = instrument
        return result
```

---

## 6. Integration Example: BAS OrderHandler

```python
# Example: BAS usage of instrument_master

from smarttrade_common.instrument_master import (
    InstrumentRegistry,
    InstrumentCache,
    InstrumentSyncService,
    InstrumentValidator,
)

class BASOrderHandler:
    def __init__(self, mds_client: BaseServiceClient, db: Database):
        # Initialize cache with DB backend
        backend = DatabaseInstrumentCacheBackend(db)
        self.cache = InstrumentCache(backend)
        
        # Initialize sync service
        self.sync_service = InstrumentSyncService(self.cache, mds_client)
        
        # Will be loaded at startup
        self.instrument_registry: Optional[InstrumentRegistry] = None
    
    async def startup(self):
        """Called at service startup."""
        # Bootstrap: load from cache or MDS
        self.instrument_registry = await self.sync_service.bootstrap()
        
        # Start background refresh job (every 6 hours)
        asyncio.create_task(self._background_refresh_job())
        
        logger.info(f"BAS started with {self.instrument_registry.size()} instruments")
    
    async def _background_refresh_job(self):
        """Periodic refresh in background."""
        while True:
            try:
                await asyncio.sleep(6 * 3600)  # 6 hours
                updated = await self.sync_service.refresh()
                if updated:
                    self.instrument_registry = self.cache.get_registry()
            except Exception as e:
                logger.error(f"Background refresh failed: {e}")
    
    async def place_order(self, order: Order) -> OrderResponse:
        """Place an order."""
        
        # Validate instrument using shared library
        result = InstrumentValidator.validate_order(
            order.symbol,
            order.quantity,
            order.price,
            self.instrument_registry,
        )
        
        if not result:
            raise OrderValidationError(f"Invalid order: {result.errors}")
        
        # All data is in-memory; no MDS calls
        # ... continue with risk validation, broker execution ...
```

---

## 7. Implementation Checklist

### Phase 1: Core Library

- [ ] Implement `models.py` (Instrument dataclass)
- [ ] Implement `cache.py` (InstrumentRegistry)
- [ ] Implement `repository.py` (InstrumentCache + backends)
- [ ] Implement `sync_service.py` (InstrumentSyncService)
- [ ] Implement `validators.py` (InstrumentValidator)
- [ ] Add unit tests (50+ tests)
- [ ] Add integration tests (MDS mock)
- [ ] Document in smarttrade_common README

### Phase 2: Service Integration

- [ ] Update BAS to use smarttrade_common.instrument_master
- [ ] Update PBS to use smarttrade_common.instrument_master
- [ ] Update Strategy Service (if applicable)
- [ ] Remove duplicate symbol logic from each service
- [ ] Add health check endpoints (`/healthz` for instrument cache)

### Phase 3: Monitoring

- [ ] Add Prometheus metrics (cache size, sync lag, failure count)
- [ ] Add alerting (cache stale >12h, sync failures >3)
- [ ] Add dashboard (instrument cache health, last sync time)
- [ ] Log staleness detection

---

## 8. Testing Strategy

### Unit Tests

```python
# test_models.py
def test_instrument_validation():
    """Tick, lot size validation."""
    instr = Instrument(
        instrument_id=UUID(...),
        symbol="NSE:INFY",
        tick_size=0.05,
        lot_size=1,
        ...
    )
    
    assert instr.validate_order_price(1500.0) == True
    assert instr.validate_order_price(1500.025) == False
    assert instr.validate_order_quantity(5) == True
    assert instr.validate_order_quantity(3) == False

# test_registry.py
def test_registry_lookup():
    """Fast O(1) lookups."""
    instrs = [Instrument(...), Instrument(...)]
    registry = InstrumentRegistry.from_instruments(instrs)
    
    assert registry.get("NSE:INFY") == instrs[0]
    assert registry.get("NSE:MISSING") is None

def test_registry_search():
    """Filtered search."""
    results = registry.search(exchange="NSE", instrument_type="EQUITY")
    assert len(results) >= 1
```

### Integration Tests

```python
# test_sync_service.py
@pytest.mark.asyncio
async def test_bootstrap_from_mds_mock(mds_mock):
    """Bootstrap with mock MDS."""
    cache = InstrumentCache(InMemoryCacheBackend())
    service = InstrumentSyncService(cache, mds_mock)
    
    registry = await service.bootstrap()
    
    assert registry.size() > 0
    assert registry.get("NSE:INFY") is not None

@pytest.mark.asyncio
async def test_refresh_idempotent(mds_mock):
    """Refresh is idempotent (no-op if unchanged)."""
    service = InstrumentSyncService(cache, mds_mock)
    
    await service.bootstrap()
    v1 = service.cache.checksum
    
    updated = await service.refresh()
    
    # No update (checksum unchanged)
    assert updated == False
    assert service.cache.checksum == v1
```

---

## 9. Deployment & Rollout

### Phase 1 (Weeks 1-2): Library Implementation
- Implement + test smarttrade_common.instrument_master
- All tests passing locally
- Code review + merge to main

### Phase 2 (Weeks 3-4): BAS Integration
- Update BAS to use shared library
- Verify orders execute correctly
- Monitor instrument cache health
- Canary deploy to staging

### Phase 3 (Weeks 5-6): Prod Rollout
- Deploy to production with feature flag
- Monitor metrics (cache size, sync lag, rejection rates)
- Gradual traffic increase (10% → 50% → 100%)
- Rollback plan: Revert to old symbol logic (temporary)

---

## 10. Metrics & Alerting

### Prometheus Metrics

```
# Gauge
instrument_cache_size{service="bas"}
instrument_cache_version{service="bas"}
instrument_sync_lag_seconds{service="bas"}
instrument_registry_lookup_time_microseconds{service="bas"}

# Counter
instrument_lookups_total{service="bas", result="hit|miss"}
instrument_sync_attempts_total{service="bas", status="success|failure"}
instrument_order_rejections_total{service="bas", reason="not_found|invalid_tick|invalid_lot"}
```

### Alerting Rules

```yaml
- alert: InstrumentCacheStale
  expr: instrument_sync_lag_seconds > 43200  # > 12 hours
  for: 1h
  action: Page on-call

- alert: InstrumentSyncFailures
  expr: rate(instrument_sync_attempts_total{status="failure"}[5m]) > 0.5
  action: Alert in #ops-alerts Slack channel
```

---

**Status**: Design Complete  
**Next**: Implementation (Phase 1, 3-4 weeks)  
**Owner**: SmartTrade Platform Team
