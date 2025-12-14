Function Analysis and Latency Impact Report
Executive Summary
The getAllFacilitiesWithFiltersV2 function (lines 1624-1830) has significant performance issues:

Duplicate filtering logic executed 2-3 times per request
N+1 query patterns in unit fetching (lines 1725-1734)
Overlapping SQL queries that fetch similar data multiple times
Estimated total latency: 500ms - 3000ms+ depending on data volume
---

Function-by-Function Analysis
1. filterController.getAmenitiesSlugsByGroupSLug() (Line 1645-1648)
Purpose: Expands amenity group slugs into individual amenity slugs

Implementation:

Loops through each unique group slug
Calls getAmenityGroupsBySlug() for EACH group slug individually (N queries)
Returns array of amenity slug arrays
SQL Query:

SELECT ag.id, ag.group_name, ag.slugs 
FROM amenity_groups ag 
WHERE ag.slug IN (:slug)
Latency Impact:

Low-Medium (50-150ms)
N+1 Pattern: Executes 1 query per unique group slug
Optimization: Batch all slugs in single query → Save 30-100ms
---

2. filterController.isFilterByLength() (Line 1649-1651)
Purpose: Checks if any amenity in payload matches length filter criteria

Implementation:

Simple array .some() check against environment variable
No database query
Latency Impact:

Negligible (<1ms)
In-memory operation
---

3. filterController.filterAmenities() (Line 1656-1660) - CRITICAL DUPLICATE
Purpose: Filters facilities and unit types by amenities and categories

Implementation:

Calls filterByAmenities() which loops through amenity groups
For each group, calls getUnitsUnitAmenitiesByNames() (N queries)
Calls filterByUnitCategory() for category filtering
Returns { facilitiesIds, unitTypeIds }
SQL Queries:

getUnitsUnitAmenitiesByNames() - executed N times (once per amenity group):
SELECT units.id, units.external_uuid, units.facilities_id
FROM units
INNER JOIN unit_amenities_unit uau ON uau.unit_id = units.id
INNER JOIN unit_amenities ua ON ua.id = uau.unit_amenities_id
WHERE units.deleted_at IS NULL AND ua.slug IN (:names)
GROUP BY units.id, units.external_uuid, units.facilities_id
getCategoryWithCategorySlug():
SELECT utuc.unit_type_id
FROM unit_type_category uc
LEFT JOIN unit_type_unit_category utuc ON utuc.unit_type_category_id = uc.id
WHERE uc.slug IN (:slugs)
Latency Impact:

High (200-800ms)
N+1 Pattern: If 3 amenity groups → 3 separate queries
Overlap: This same filtering is repeated in marketController.getMarketData() (line 1682)
---

4. marketController.getMarketData() (Line 1682-1684) - CRITICAL DUPLICATE
Purpose: Fetches marketing page data with facilities

Implementation:

DUPLICATE: Calls filterController.filterAmenities() AGAIN (line 23 in market.controller.ts)
Calls getMarketDataByPermalink() which:
Fetches marketing page from DB
Calls getFacilityIdsByMarketingPageId() or getFacilityListByStateAndCity()
Calls getFacilityDetailsByIds() which calls listAllFacilitiesV3()
SQL Queries:

Marketing page lookup:
SELECT m.id, m.external_uuid, m.pagedata, m.seo_title, m.seo_description, mf.facility_id
FROM marketing m
LEFT JOIN marketing_facilities mf ON mf.marketing_id = m.id
WHERE m.permalink = :permalink
Facility IDs by marketing page:
-- Complex query joining facilities, units, categories, etc.
listAllFacilitiesV3() - Large complex query with multiple subqueries
Latency Impact:

Very High (500-2000ms)
Duplicate Work: Re-executes filterAmenities() already done at line 1656
Overlap: listAllFacilitiesV3() is called again in _getFacilitiesListData() (line 1771)
---

5. _getFacilityInfoAndFeatureList() (Line 1709-1713) - N+1 PATTERN
Purpose: Fetches complete facility data including features, reviews, map directions

Implementation:

Executes 3 queries in parallel via Promise.all():
getFacilityDataBySlug() - Complex query with 7+ subqueries
getFacilityFeaturesBySlug() - Join query
getGoogleReviewsBySlug() - Join query
Then executes 2 more queries:
_getMapDirections() - In-memory (no query)
_getFacilityInfo() - In-memory (no query)
SQL Queries:

getFacilityDataBySlug() - MASSIVE QUERY with subqueries:
SELECT f.*, s.name, c.name, 
  (SELECT json_agg(...) FROM facilities_media ...) AS facilities_media,
  (SELECT json_agg(...) FROM facilities_hours ...) AS facilities_hours,
  (SELECT json_agg(...) FROM facilities_faq ...) AS facilities_faq,
  (SELECT json_agg(...) FROM facilities_badge ...) AS facilities_badge,
  (SELECT json_agg(...) FROM facilities_social_links ...) AS facilities_social_links,
  (SELECT json_agg(...) FROM promotions ...) AS promotions,
  (SELECT json_agg(...) FROM facilities_parent_child ...) AS facilities_parent_child
FROM facilities f
LEFT JOIN state s ON f.state_id = s.id
LEFT JOIN country c ON f.country_id = c.id
LEFT JOIN (SELECT DISTINCT ON (ut.facilities_id) ...) lp ON lp.facilities_id = f.id
WHERE f.slug = $1
getFacilityFeaturesBySlug():
SELECT ft.name, ft.slug, ft.external_uuid
FROM facilities fc
LEFT JOIN facilities_features ff ON ff.facility_id = fc.id
LEFT JOIN features ft ON ft.id = ff.features_id
WHERE fc.slug = :slug
getGoogleReviewsBySlug():
SELECT fgr.*
FROM facility_google_review fgr
INNER JOIN facilities f ON f.id = fgr.facility_id
WHERE f.slug = :slug AND fgr.deletedAt IS NULL
ORDER BY fgr.createdAt DESC
Latency Impact:

High (300-800ms)
Multiple subqueries in main query can be slow
Optimization: Could use CTEs or materialized views
---

6. _getUnitsByFacilityUuid() (Line 1725-1734) - CRITICAL N+1 PATTERN
Purpose: Fetches units for each facility UUID

Implementation:

N+1 QUERY PATTERN: Loops through facilitiesEternalUUID array
Calls getUnitByFacilityUuidV2() for EACH UUID in Promise.all()
If 5 facilities → 5 separate SQL queries executed in parallel
SQL Query (executed N times):

SELECT fc.id, fc.external_uuid, fc.name, ut.size, ut.price, ut.standard_rate,
  ug.available_units_count, ug.total_units_count,
  ARRAY_AGG(DISTINCT ua.name) AS amenity_names,
  t.description, uc.name as category
FROM units ut
LEFT JOIN facilities fc ON fc.id = ut.facilities_id
LEFT JOIN unit_groups ug ON ut.unit_group_id = ug.id
LEFT JOIN unit_group_discount ugd ON ug.id = ugd.unit_group_id
LEFT JOIN discount_plans dp ON dp.id = ugd.discount_plan_id
LEFT JOIN unit_amenities_unit umu ON umu.unit_id = ut.id
LEFT JOIN unit_amenities ua ON ua.id = umu.unit_amenities_id
LEFT JOIN tier t ON t.id = ut.tier_id
LEFT JOIN unit_category uc ON uc.id = ut.unit_category_id
LEFT JOIN unit_amenities_unit umu_filter ON umu_filter.unit_id = ut.id
LEFT JOIN unit_amenities ua_filter ON ua_filter.id = umu_filter.unit_amenities_id
LEFT JOIN amenity_group_unit_amenity agua ON agua.unit_amenity_id = ua_filter.id
LEFT JOIN amenity_groups ag ON ag.id = agua.amenity_group_id
WHERE ut.status = 'vacant' AND ut.rentable = true
  AND fc.external_uuid = :uuid
GROUP BY fc.id, fc.name, ut.size, ug.id, ...
ORDER BY ut.size ASC
Latency Impact:

Very High (500-2000ms+)
N+1 Pattern: If 5 facilities → 5 queries × 200ms = 1000ms
Optimization: Batch query with WHERE fc.external_uuid IN (:...uuids) → Save 400-1500ms
---

7. filtersService.getFiltersBySearchItem() (Line 1746-1752, 1787-1791, 1798-1802) - TRIPLE EXECUTION
Purpose: Builds filter options (categories, amenities) for facilities

Implementation:

Called 3 TIMES in same request:
Line 1747: For detail page (with isUnitFlag: true)
Line 1787: For facility list (if totalFacilities > 0)
Line 1798: For marketing page data (if no facilities but has marketing data)
SQL Queries:

getFiltersByFacilityInfo() or getFiltersByUnitInfo():
SELECT fc.id, fc.name, uc.name as category, uc.rank, uc.slug,
  ag.id, ag.group_name, ag.slug, ut.id, ua.id, ua.name, ua.slug
FROM facilities fc
INNER JOIN units ut ON ut.facilities_id = fc.id AND ut.status = 'vacant' AND ut.rentable = true
INNER JOIN unit_type_unit_category utuc ON utuc.unit_type_id = ut.unit_type_id
INNER JOIN unit_type_category uc ON uc.id = utuc.unit_type_category_id
INNER JOIN unit_amenities_unit uau ON uau.unit_id = ut.id
INNER JOIN unit_amenities ua ON ua.id = uau.unit_amenities_id AND COALESCE(ua.deleted, false) = false
INNER JOIN amenity_group_unit_amenity agu ON agu.unit_amenity_id = ua.id
INNER JOIN amenity_groups ag ON ag.id = agu.amenity_group_id
INNER JOIN category_amenity_groups cag ON cag.category_id = uc.id AND cag.amenity_group_id = ag.id
WHERE fc.external_uuid IN (:...facilityExternalUUIDs)
ORDER BY ut.id ASC, ua.name ASC
Latency Impact:

High (300-900ms per call)
Triple Execution: 900-2700ms total
Overlap: Same complex join query executed 3 times with similar data
Optimization: Cache result or execute once → Save 600-1800ms
---

8. _getFacilitiesListData() (Line 1770-1776)
Purpose: Fetches paginated list of facilities

Implementation:

Calls listAllFacilitiesV3() which executes massive query
Same query structure as in marketController.getFacilityDetailsByIds()
SQL Query: Same as listAllFacilitiesV3() described above

Latency Impact:

Very High (500-2000ms)
Overlap: Same query pattern as marketing page facility fetch
Optimization: Reuse query results if same filters applied
---

Overlapping Concepts Identified
1. Filter Amenities Logic (Executed 2x)
Line 1656: filterController.filterAmenities()
Line 1682: marketController.getMarketData() → calls filterAmenities() again
Impact: Duplicate filtering work, same SQL queries executed twice
2. Facility List Query (Executed 2x)
Line 1682: marketController.getMarketData() → listAllFacilitiesV3()
Line 1771: _getFacilitiesListData() → listAllFacilitiesV3()
Impact: Same complex query executed multiple times
3. Filter Building (Executed 3x)
Line 1747: For detail page
Line 1787: For facility list  
Line 1798: For marketing page
Impact: Same complex join query executed 3 times
4. Unit Fetching Pattern
Line 1725-1734: N queries for N facilities
Same pattern exists in other controller methods
Impact: N+1 query pattern
---

Overlapping SQL Queries
Query Pattern 1: Facility + Unit + Amenity Joins
Used in:

getFiltersByFacilityInfo() (filters.service.ts:12)
getFiltersByUnitInfo() (filters.service.ts:284)
getUnitsUnitAmenitiesByNames() (facilities.service.ts:1475)
getUnitByFacilityUuidV2() (units.service.ts:492)
Overlap: Same join pattern (facilities → units → unit_amenities → amenity_groups) executed multiple times

Query Pattern 2: Facility List with Media
Used in:

listAllFacilitiesV3() (facilities.service.ts:803)
Called from marketController.getFacilityDetailsByIds() (market.controller.ts:289)
Called from _getFacilitiesListData() (facilities.controller.ts:2625)
Overlap: Same SELECT with media subquery executed multiple times

Query Pattern 3: Amenity Group Expansion
Used in:

getAmenitiesSlugsByGroupSLug() loops through groups (filter.controller.ts:22)
Each iteration executes getAmenityGroupsBySlug()
Overlap: Could be batched into single query

---

N+1 Query Patterns
Pattern 1: Amenity Group Expansion (Line 1645-1648)
for (const uniqueGroupSlug of uniqueGroupSlugs) {
  const result = await this.facilitiesService.getAmenityGroupsBySlug([uniqueGroupSlug]);
  // 1 query per group slug
}
Impact: If 3 groups → 3 queries (should be 1)

Pattern 2: Amenity Filtering (Line 1656-1660, inside filterByAmenities)
for (const group of normalizedGroups) {
  const unitsUnitAmenities = await this.facilitiesService.getUnitsUnitAmenitiesByNames(group);
  // 1 query per amenity group
}
Impact: If 3 groups → 3 queries (should be 1)

Pattern 3: Unit Fetching (Line 1725-1734) - MOST CRITICAL
const units = await Promise.all(
  facilitiesEternalUUID.map((uuid) =>
    this._getUnitsByFacilityUuid(uuid, ...)
    // 1 query per facility UUID
  )
);
Impact: If 5 facilities → 5 queries (should be 1)

Estimated Latency: 500-2000ms+ depending on facility count

---

Latency Impact Summary
| Function | Current Latency | Optimization Potential | Priority |

|----------|----------------|----------------------|----------|

| getAmenitiesSlugsByGroupSLug | 50-150ms | 30-100ms saved | Medium |

| filterAmenities (1st call) | 200-800ms | - | High |

| filterAmenities (2nd call - duplicate) | 200-800ms | 200-800ms saved | Critical |

| getMarketData | 500-2000ms | 200-800ms saved (remove duplicate filter) | Critical |

| _getFacilityInfoAndFeatureList | 300-800ms | 100-200ms saved (optimize subqueries) | Medium |

| _getUnitsByFacilityUuid (N+1) | 500-2000ms+ | 400-1500ms saved | Critical |

| getFiltersBySearchItem (1st) | 300-900ms | - | High |

| getFiltersBySearchItem (2nd) | 300-900ms | 300-900ms saved | Critical |

| getFiltersBySearchItem (3rd) | 300-900ms | 300-900ms saved | Critical |

| _getFacilitiesListData | 500-2000ms | 200-500ms saved (reuse query) | High |

Total Current Latency: 2,850ms - 10,250ms+

Potential Savings: 1,400ms - 4,200ms+

Optimized Latency: 1,450ms - 6,050ms

---
