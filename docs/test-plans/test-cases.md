# Test Plan

## LD Relay Advisor

---

## 1. Decision Engine Tests

### 1.1 Relay Proxy Needed

| ID | Scenario | Input | Expected Output | Source |
|----|----------|-------|-----------------|--------|
| R01 | On-prem hosting | hosting=on-prem | needs_relay=True | Common pattern: ARC, ACI |
| R02 | Hybrid hosting | hosting=hybrid | needs_relay=True | Common pattern |
| R03 | Cloud with direct access, few services | hosting=aws, internet=direct, services=<10 | needs_relay=False | Tricon-like profile |
| R04 | Cloud with proxy/firewall | hosting=aws, internet=proxy-firewall | needs_relay=True | Network constraint |
| R05 | Air-gapped | air_gapped=True | needs_relay=True | ACI offline pattern |
| R06 | Many services, cloud, direct | hosting=aws, internet=direct, services=200+ | needs_relay=True | Scale-driven |
| R07 | Ad-blocker concern | adblocker_concern=True | needs_relay=True | NinjaTrader pattern |
| R08 | Customer-hosted | hosting=customer-hosted | needs_relay=True | ACI pattern |

### 1.2 Mode Recommendation

| ID | Scenario | Input | Expected Mode | Reason |
|----|----------|-------|---------------|--------|
| M01 | Has browser apps | has_browser_apps=True | proxy | Daemon can't serve browser SDKs |
| M02 | Server-only + PHP | has_browser_apps=False, languages=[php] | daemon | PHP can't do streaming |
| M03 | Air-gapped | air_gapped=True | offline | Only option for no-internet |
| M04 | Server-only, centralized | has_browser_apps=False, pref=centralized | proxy | Preference alignment |
| M05 | Server-only, no preference | has_browser_apps=False, pref=embedded | proxy | Safer default |
| M06 | ARC profile | has_browser_apps=True, hosting=on-prem | proxy | Browser apps + on-prem |
| M07 | Island profile | has_browser_apps=True, hosting=aws, k8s | proxy | Browser-heavy |

### 1.3 Redis Recommendation

| ID | Scenario | Input | Expected | Reason |
|----|----------|-------|----------|--------|
| D01 | Daemon mode | mode=daemon | needs_redis=True | Required for daemon |
| D02 | Zero downtime | outage_tolerance=zero-downtime | needs_redis=True | Resilience |
| D03 | Frequent deploys | frequent_deployments=True | needs_redis=True | Cold-start |
| D04 | Fallback OK | outage_tolerance=fallback-ok, deploys=False | needs_redis=False | Optional |
| D05 | Shared Redis + PCI | redis_shared=True, compliance=pci | separate_instance=True | ARC pattern |
| D06 | Shared Redis, no compliance | redis_shared=True, compliance=none | separate_instance=True | Eviction conflict |
| D07 | No existing Redis | existing_datastore=none | separate_instance=N/A | New instance |

## 2. Sizing Calculator Tests

| ID | Scenario | Input | Expected | Validation |
|----|----------|-------|----------|------------|
| S01 | Minimum deployment | any profile | instances >= 3 | LD official minimum |
| S02 | Small scale | connections=<100 | instances = 3 | Min 3 even for small |
| S03 | Large scale | connections=10000+ | instances > 3 | Scaled up |
| S04 | CPU always 4 | any | vcpu = 4 | Matches m4.xlarge |
| S05 | Memory 8 GB | any | memory = 8 GB | Our starting recommendation |
| S06 | Redis eviction | any | eviction = noeviction | Always |
| S07 | Redis TTL | any | cache_ttl = infinite (-1) | LD recommendation |
| S08 | Zero downtime Redis | outage=zero-downtime | persistence = AOF | Recommended |
| S09 | Fallback OK Redis | outage=fallback-ok | persistence = optional | Not required |

## 3. Config Generator Tests

| ID | Test | Validation |
|----|------|------------|
| C01 | Proxy mode config | Valid TOML, has [Main], [Environment], port=8030 |
| C02 | Proxy mode + Redis | Has [Redis] section with host, localTtl |
| C03 | Daemon mode config | Valid TOML, has appropriate daemon settings |
| C04 | Proxy config | Has [Proxy] section when internet_access=proxy-firewall |
| C05 | Multiple environments | Has [Environment "Prod"] and [Environment "Staging"] |
| C06 | Infinite TTL | localTtl = "-1" when zero-downtime |

## 4. Report Generator Tests

| ID | Test | Validation |
|----|------|------------|
| P01 | PDF generates | Output file exists and is valid PDF |
| P02 | Company name in title | PDF contains client company name |
| P03 | Recommendation matches | PDF recommendation matches decision engine output |
| P04 | Sizing table present | PDF has sizing table with correct numbers |
| P05 | References section | PDF has links to official LD docs |
| P06 | No LD-unofficial claims without label | Every estimate is labeled as estimate |

## 5. End-to-End Scenario Tests

Run real client profiles through the full pipeline and validate against what the SA actually recommended.

### Scenario: ARC Airlines

```
Input:
  hosting: on-prem
  internet_access: proxy-firewall
  has_browser_apps: true (Cloudflare UI)
  has_server_apps: true
  service_count: 50-200
  environment_count: 4
  outage_tolerance: zero-downtime
  frequent_deployments: true
  infra_preference: centralized
  existing_datastore: redis
  redis_shared: true
  redis_compliance: pci
  compliance: [pci]
  security_review_needed: true
  peak_connections: 100-1000

Expected:
  needs_relay: True
  mode: proxy
  needs_redis: True
  redis_separate: True
  redis_persistence: True (AOF)
  instances: 3
```

### Scenario: Island Browser

```
Input:
  hosting: aws
  internet_access: direct
  has_browser_apps: true
  has_server_apps: true
  service_count: 50-200
  environment_count: 3
  outage_tolerance: acceptable-degradation
  container_orchestration: kubernetes
  peak_connections: 10000+

Expected:
  needs_relay: True (scale + browser security)
  mode: proxy
  needs_redis: False (acceptable degradation)
  instances: 3+ (scale dependent)
```

### Scenario: NinjaTrader (no relay today)

```
Input:
  hosting: aws
  internet_access: direct
  has_browser_apps: true
  has_server_apps: true
  service_count: <10
  adblocker_concern: true

Expected:
  needs_relay: True (ad-blocker concern)
  mode: proxy
```

### Scenario: Small Cloud Startup (no relay needed)

```
Input:
  hosting: aws
  internet_access: direct
  has_browser_apps: true
  has_server_apps: true
  service_count: <10
  outage_tolerance: fallback-ok
  adblocker_concern: false
  compliance: none

Expected:
  needs_relay: False
  mode: not-needed
```

## 6. RAG Chat Tests

| ID | Question | Expected Behavior |
|----|----------|-------------------|
| Q01 | "What happens if Redis goes down during an LD outage?" | Answer references cache layers, cites codebase |
| Q02 | "Can I use a separate Redis logical DB?" | Answer says yes but recommends separate instance, cites util_test.go |
| Q03 | "What does the cache TTL do?" | Whiteboard analogy or equivalent, cites LD docs |
| Q04 | "How do Big Segments work?" | Explains individual membership storage, cites bigsegments/ |
| Q05 | "What ports do I need to open?" | Lists stream/sdk/events endpoints, cites config.go |
| Q06 | Made-up question about nonexistent feature | Responds "I cannot verify this in the documentation" |
