# Performance Issues - Complete Functions List by Service

This document lists all functions with performance issues across all services, organized by service and issue type.

---

## 📋 Summary

**Total Functions with Issues: 24**

### By Service:
- **[service-facilities](#service-facilities-18-issues):** 18 functions
- **[service-workers](#service-workers-5-issues):** 5 functions  
- **[service-cms](#service-cms-1-issue):** 1 function
- **[service-auth](#service-auth-0-issues):** 0 functions
- **[service-crons](#service-crons-0-issues):** 0 functions
- **[service-issues](#service-issues-0-issues):** 0 functions
- **[service-payments](#service-payments-0-issues):** 0 functions
- **[service-pms](#service-pms-0-issues):** 0 functions
- **[service-tenants](#service-tenants-0-issues):** 0 functions
- **[service-users](#service-users-0-issues):** 0 functions

### By Issue Type:
- **N+1 Query Patterns:** 11 functions
- **Duplicate Function Calls:** 3 functions
- **Overlapping SQL Queries:** 4 functions
- **Improper Async/Await:** 5 functions
- **Multiple Query Executions:** 1 function

---

## service-facilities (18 Issues)

### Controllers

#### 1. `filterController.getAmenitiesSlugsByGroupSLug()`
- **File:** `service-facilities/src/controllers/filter.controller.ts:14-34`
- **Issue Type:** N+1 Query Pattern
- **Problem:** Loops through unique group slugs, calls `getAmenityGroupsBySlug()` individually for each slug
- **Impact:** Low-Medium (50-150ms)
- **Fix:** Batch all slugs in single query → Save 30-100ms

#### 2. `filterController.filterByAmenities()`
- **File:** `service-facilities/src/controllers/filter.controller.ts:62-107`
- **Issue Type:** N+1 Query Pattern
- **Problem:** Loops through normalized amenity groups, calls `getUnitsUnitAmenitiesByNames()` for each group
- **Impact:** High (200-800ms)
- **Fix:** Batch all groups in single query → Save 100-500ms

#### 3. `facilitiesController.getAllFacilitiesWithFiltersV2()` - Unit Fetching
- **File:** `service-facilities/src/controllers/facilities.controller.ts:1726-1735`
- **Issue Type:** N+1 Query Pattern
- **Problem:** Uses Promise.all with map to call `_getUnitsByFacilityUuid()` for each facility UUID
- **Impact:** Very High (500-2000ms+)
- **Fix:** Batch query with WHERE fc.external_uuid IN (:...uuids) → Save 400-1500ms

#### 4. `facilitiesController.getFacilityBySlug()` - Unit Fetching
- **File:** `service-facilities/src/controllers/facilities.controller.ts:581-590`
- **Issue Type:** N+1 Query Pattern
- **Problem:** Same N+1 pattern - calls `_getUnitsByFacilityUuid()` for each facility UUID
- **Impact:** Very High (500-2000ms+)
- **Fix:** Batch query → Save 400-1500ms

#### 5. `facilitiesController.getFacilityBySlugV2()` - Unit Fetching
- **File:** `service-facilities/src/controllers/facilities.controller.ts:688-697`
- **Issue Type:** N+1 Query Pattern
- **Problem:** Same N+1 pattern - calls `_getUnitsByFacilityUuid()` for each facility UUID
- **Impact:** Very High (500-2000ms+)
- **Fix:** Batch query → Save 400-1500ms

#### 6. `facilitiesController.getAllFacilitiesWithFiltersV2()` - filterAmenities Duplicate
- **File:** `service-facilities/src/controllers/facilities.controller.ts:1658-1683`
- **Issue Type:** Duplicate Function Call
- **Problem:** Calls `filterController.filterAmenities()` at line 1658, then `marketController.getMarketData()` calls it again at line 23
- **Impact:** Critical - 200-800ms wasted
- **Fix:** Pass already filtered results to getMarketData → Save 200-800ms

#### 7. `facilitiesController.getAllFacilitiesWithFiltersV2()` - getFiltersBySearchItem Triple Call
- **File:** `service-facilities/src/controllers/facilities.controller.ts:1747-1803`
- **Issue Type:** Duplicate Function Call
- **Problem:** Calls `filtersService.getFiltersBySearchItem()` three times (lines 1747, 1788, 1799)
- **Impact:** Critical - 600-1800ms wasted (300-900ms per call)
- **Fix:** Cache result or execute once → Save 600-1800ms

#### 8. `facilitiesController.getAllFacilitiesWithFiltersV2()` - listAllFacilitiesV3 Duplicate
- **File:** `service-facilities/src/controllers/facilities.controller.ts:1683, 1771`
- **Issue Type:** Duplicate Function Call
- **Problem:** `listAllFacilitiesV3()` called in `marketController.getMarketData()` and again in `_getFacilitiesListData()`
- **Impact:** High - 200-500ms wasted
- **Fix:** Reuse query results if same filters applied → Save 200-500ms

#### 9. `facilitiesController._getFacilityInfoAndFeatureList()`
- **File:** `service-facilities/src/controllers/facilities.controller.ts:2978-3025`
- **Issue Type:** Multiple Query Executions
- **Problem:** Executes multiple complex queries in parallel (getFacilityDataBySlug with 7+ subqueries, getFacilityFeaturesBySlug, getGoogleReviewsBySlug)
- **Impact:** High (300-800ms)
- **Fix:** Use CTEs or materialized views for subqueries → Save 100-200ms
- **Called from:** `getFacilityBySlug()`, `getFacilityBySlugV2()`, `getAllFacilitiesWithFiltersV2()`

#### 10. `adminController._insertTimeTableOfficeHoursData()`
- **File:** `service-facilities/src/admin/admin.controller.ts:1655-1674`
- **Issue Type:** Improper Async/Await Pattern
- **Problem:** Uses `forEach` with async callback - doesn't wait for completion
- **Impact:** Medium - Operations may not complete before function returns
- **Fix:** Use `for...of` loop or `Promise.all()` with `map()`

#### 11. `adminController._insertTimeTableAccessHoursData()`
- **File:** `service-facilities/src/admin/admin.controller.ts:1676-1693`
- **Issue Type:** Improper Async/Await Pattern
- **Problem:** Uses `forEach` with async callback - doesn't wait for completion
- **Impact:** Medium - Operations may not complete
- **Fix:** Use proper async iteration

#### 12. `adminController._insertFaqData()`
- **File:** `service-facilities/src/admin/admin.controller.ts:1695-1707`
- **Issue Type:** Improper Async/Await Pattern
- **Problem:** Uses `forEach` with async callback - doesn't wait for completion
- **Impact:** Medium - Operations may not complete
- **Fix:** Use proper async iteration

#### 13. `adminController._insertBadgeData()`
- **File:** `service-facilities/src/admin/admin.controller.ts:1719-1732`
- **Issue Type:** Improper Async/Await Pattern
- **Problem:** Uses `forEach` with async callback - doesn't wait for completion
- **Impact:** Medium - Operations may not complete
- **Fix:** Use proper async iteration

#### 14. `adminController._insertPromotionData()`
- **File:** `service-facilities/src/admin/admin.controller.ts:1734-1744`
- **Issue Type:** Improper Async/Await Pattern
- **Problem:** Uses `forEach` with async callback - doesn't wait for completion
- **Impact:** Medium - Operations may not complete
- **Fix:** Use proper async iteration

### Services

#### 15. `filtersService.getFiltersByFacilityInfo()`
- **File:** `service-facilities/src/services/filters.service.ts:7-78`
- **Issue Type:** Overlapping SQL Query
- **Problem:** Complex join query (facilities → units → unit_amenities → amenity_groups) executed multiple times
- **Impact:** High (300-900ms per call)
- **Fix:** Cache results or batch operations
- **Called from:** `filtersService.getFiltersBySearchItem()` (line 186)

#### 16. `filtersService.getFiltersByUnitInfo()`
- **File:** `service-facilities/src/services/filters.service.ts:284-355`
- **Issue Type:** Overlapping SQL Query
- **Problem:** Similar complex join query pattern executed multiple times
- **Impact:** High (300-900ms per call)
- **Fix:** Cache results or batch operations
- **Called from:** `filtersService.getFiltersBySearchItem()` (line 174)

#### 17. `facilitiesService.listAllFacilitiesV3()`
- **File:** `service-facilities/src/services/facilities.service.ts:803`
- **Issue Type:** Overlapping SQL Query
- **Problem:** Large complex query with multiple subqueries executed multiple times
- **Impact:** Very High (500-2000ms per call)
- **Fix:** Reuse query results, implement caching
- **Called from:** `marketController.getFacilityDetailsByIds()`, `facilitiesController._getFacilitiesListData()`, `facilitiesController.getAllFacilitiesWithFiltersV2()`

#### 18. `unitsService.getUnitByFacilityUuidV2()`
- **File:** `service-facilities/src/services/units.service.ts:492`
- **Issue Type:** Overlapping SQL Query
- **Problem:** Complex query with multiple joins executed N times (once per facility)
- **Impact:** Very High (200-400ms per call × N facilities)
- **Fix:** Batch query with WHERE fc.external_uuid IN (:...uuids)
- **Called from:** `facilitiesController._getUnitsByFacilityUuid()` (line 3112)

---

## service-workers (5 Issues)

### Handlers

#### 19. `unitsSyncHandler.createUnitsPayload()` - calculateSize N+1
- **File:** `service-workers/src/handlers/unitsSync.handler.ts:160-209`
- **Issue Type:** N+1 Query Pattern
- **Problem:** Loops through units, calls `calculateSize()` which queries `unitCategoryService.getUnitCategoryIdByName()` for each unit
- **Impact:** Medium-High (50-200ms per unit)
- **Fix:** Batch category lookups or cache category mappings → Save 30-150ms per unit

#### 20. `facilitiesSyncHandler.createFacilitiesPayload()` - State/Country Lookup
- **File:** `service-workers/src/handlers/facilitiesSync.handler.ts:110-133`
- **Issue Type:** N+1 Query Pattern
- **Problem:** Loops through facilities, calls `getStateId()` and `getCountryId()` individually for each facility
- **Impact:** Medium (100-300ms per facility)
- **Fix:** Batch all state/country IDs in single queries → Save 50-200ms per facility

#### 21. `facilitiesSyncHandler.createAccessHoursPayload()` - Facility Lookup N+1
- **File:** `service-workers/src/handlers/facilitiesSync.handler.ts:152-185`
- **Issue Type:** N+1 Query Pattern
- **Problem:** Loops through hours data, calls `getFacilityIdByPmsFacilityId()` for each facility
- **Impact:** Medium-High (50-150ms per facility)
- **Fix:** Batch facility lookups with WHERE pms_facility_id IN (:...ids) → Save 30-100ms per facility

#### 22. `facilitiesSyncHandler.createOfficeHoursPayload()` - Facility Lookup N+1
- **File:** `service-workers/src/handlers/facilitiesSync.handler.ts:187-220`
- **Issue Type:** N+1 Query Pattern
- **Problem:** Same N+1 pattern - calls `getFacilityIdByPmsFacilityId()` for each facility
- **Impact:** Medium-High (50-150ms per facility)
- **Fix:** Batch facility lookups → Save 30-100ms per facility

#### 23. `unitGroupsSyncHandler.createUnitGroupsPayload()` - Reservation Fees N+1
- **File:** `service-workers/src/handlers/unitGroupsSync.handler.ts:154-199`
- **Issue Type:** N+1 Query Pattern
- **Problem:** Loops through unit groups, calls `getReservationFeesDetailsByPMSId()` for each reservation fee
- **Impact:** Medium (50-150ms per unit group)
- **Fix:** Batch reservation fee lookups → Save 30-100ms per unit group

---

## service-cms (1 Issue)

### Services

#### 24. `blogsService.createBlog()` - Category Lookup N+1
- **File:** `service-cms/src/blogs/blogs.service.ts:54-119`
- **Issue Type:** N+1 Query Pattern
- **Problem:** Loops through category IDs, calls `categoryRepository.findOneBy()` for each category
- **Impact:** Medium (50-150ms per category)
- **Fix:** Batch category lookups with WHERE id IN (:...ids) → Save 30-100ms per category

---

## service-auth (0 Issues)

_No performance issues identified yet._

---

## service-crons (0 Issues)

_No performance issues identified yet._

---

## service-issues (0 Issues)

_No performance issues identified yet._

---

## service-payments (0 Issues)

_No performance issues identified yet._

---

## service-pms (0 Issues)

_No performance issues identified yet._

---

## service-tenants (0 Issues)

_No performance issues identified yet._

---

## service-users (0 Issues)

_No performance issues identified yet._

---

## Detailed Issue Breakdown

### N+1 Query Patterns (11 functions)

1. ✅ `filterController.getAmenitiesSlugsByGroupSLug()` - service-facilities
2. ✅ `filterController.filterByAmenities()` - service-facilities
3. ✅ `facilitiesController.getAllFacilitiesWithFiltersV2()` - Unit Fetching - service-facilities
4. ✅ `facilitiesController.getFacilityBySlug()` - Unit Fetching - service-facilities
5. ✅ `facilitiesController.getFacilityBySlugV2()` - Unit Fetching - service-facilities
6. ✅ `unitsSyncHandler.createUnitsPayload()` - service-workers
7. ✅ `facilitiesSyncHandler.createFacilitiesPayload()` - service-workers
8. ✅ `facilitiesSyncHandler.createAccessHoursPayload()` - service-workers
9. ✅ `facilitiesSyncHandler.createOfficeHoursPayload()` - service-workers
10. ✅ `unitGroupsSyncHandler.createUnitGroupsPayload()` - service-workers
11. ✅ `blogsService.createBlog()` - service-cms

### Duplicate Function Calls (3 functions)

1. ✅ `facilitiesController.getAllFacilitiesWithFiltersV2()` - filterAmenities Duplicate - service-facilities
2. ✅ `facilitiesController.getAllFacilitiesWithFiltersV2()` - getFiltersBySearchItem Triple Call - service-facilities
3. ✅ `facilitiesController.getAllFacilitiesWithFiltersV2()` - listAllFacilitiesV3 Duplicate - service-facilities

### Overlapping SQL Queries (4 functions)

1. ✅ `filtersService.getFiltersByFacilityInfo()` - service-facilities
2. ✅ `filtersService.getFiltersByUnitInfo()` - service-facilities
3. ✅ `facilitiesService.listAllFacilitiesV3()` - service-facilities
4. ✅ `unitsService.getUnitByFacilityUuidV2()` - service-facilities

### Improper Async/Await Patterns (5 functions)

1. ✅ `adminController._insertTimeTableOfficeHoursData()` - service-facilities
2. ✅ `adminController._insertTimeTableAccessHoursData()` - service-facilities
3. ✅ `adminController._insertFaqData()` - service-facilities
4. ✅ `adminController._insertBadgeData()` - service-facilities
5. ✅ `adminController._insertPromotionData()` - service-facilities

### Multiple Query Executions (1 function)

1. ✅ `facilitiesController._getFacilityInfoAndFeatureList()` - service-facilities

---

## 🎯 Priority Recommendations

### 🔴 Critical Priority:
1. Fix duplicate `filterAmenities()` call in `getAllFacilitiesWithFiltersV2()` - service-facilities
2. Fix triple `getFiltersBySearchItem()` calls in `getAllFacilitiesWithFiltersV2()` - service-facilities
3. Batch `_getUnitsByFacilityUuid()` calls (N+1 pattern) - service-facilities

### 🟠 High Priority:
4. Batch `filterByAmenities()` loop queries - service-facilities
5. Reuse `listAllFacilitiesV3()` results - service-facilities
6. Fix forEach async patterns in admin controller - service-facilities
7. Batch facility lookups in `facilitiesSyncHandler` methods - service-workers

### 🟡 Medium Priority:
8. Batch `getAmenitiesSlugsByGroupSLug()` queries - service-facilities
9. Optimize `_getFacilityInfoAndFeatureList()` subqueries - service-facilities
10. Batch category lookups in `blogsService.createBlog()` - service-cms
11. Batch reservation fee lookups in `unitGroupsSyncHandler` - service-workers
12. Batch state/country lookups in `facilitiesSyncHandler.createFacilitiesPayload()` - service-workers
13. Optimize `unitsSyncHandler.calculateSize()` to batch category lookups - service-workers

---

## 📈 Estimated Impact

### Current Total Latency:
- **service-facilities:** 2,850ms - 10,250ms+ per request (for getAllFacilitiesWithFiltersV2)
- **service-workers:** 200-800ms per sync operation
- **service-cms:** 50-150ms per blog creation

### Potential Savings:
- **service-facilities:** 1,400ms - 4,200ms+
- **service-workers:** 100-500ms per sync
- **service-cms:** 30-100ms per blog

### Optimized Latency:
- **service-facilities:** 1,450ms - 6,050ms
- **service-workers:** 100-300ms per sync
- **service-cms:** 20-50ms per blog

---

## 🔍 Code Examples

### N+1 Pattern Example:
```typescript
// ❌ BAD: N queries
for (const item of items) {
  const result = await service.getById(item.id);
}

// ✅ GOOD: 1 query
const ids = items.map(item => item.id);
const results = await service.getByIds(ids);
```

### Duplicate Call Example:
```typescript
// ❌ BAD: Called twice
const result1 = await filterService.filter(data);
const result2 = await otherService.process(data); // calls filterService.filter() again

// ✅ GOOD: Pass result
const result1 = await filterService.filter(data);
const result2 = await otherService.process(data, result1);
```

### forEach Async Example:
```typescript
// ❌ BAD: Doesn't wait
items.forEach(async (item) => {
  await service.save(item);
});

// ✅ GOOD: Waits for all
await Promise.all(items.map(item => service.save(item)));
// OR
for (const item of items) {
  await service.save(item);
}
```
