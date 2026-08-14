# Azure VM Instance Types & Free Tier Cheatsheet

> Sources: [Azure VM sizes overview](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview), [Azure free services](https://azure.microsoft.com/en-au/pricing/free-services), [Free account services](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/create-free-services), [Azure regions list](https://learn.microsoft.com/en-us/azure/reliability/regions-list)

---

## VM Naming Convention

```
[Family][Subfamily][vCPUs][Features][Version]

Example: Standard_D8as_v5
         D = Family (General Purpose)
         8 = vCPUs
         a = AMD CPU
         s = Premium Storage capable
         v5 = Version 5
```

### Feature Suffixes

| Letter | Meaning |
|--------|---------|
| `a` | AMD CPU |
| `p` | ARM-based (Cobalt/Ampere) |
| `d` | Local temp disk |
| `s` | Premium Storage capable |
| `l` | Low memory (2 GiB/vCPU) |
| `m` | High memory |
| `t` | Tiny memory |
| `i` | Isolated |
| `b` | Block storage performance |
| `C` | Confidential |

---

## VM Family Categories

### General Purpose (Balanced CPU-to-Memory)

| Family | Series | Use Cases | vCPU Range | Notes |
|--------|--------|-----------|------------|-------|
| **A-family** | Av2 | Dev/test, entry-level workloads, low-traffic web servers | 1-8 | Most economical, no premium storage |
| **B-family** | Bsv2, Basv2, Bpsv2 | Burstable workloads, dev/test, small DBs, low-traffic web | 1-32 | CPU credit model, cheapest option |
| **D-family** | Dsv7/Ddsv7, Dasv7/Dadsv7, Dpsv6, Dsv6/Ddsv6, Dv5/Dsv5, Dasv5/Dadsv5 | Enterprise apps, relational DBs, in-memory caching, analytics | 2-160 | Balanced performance, most popular |
| **DC-family** | DCasv5/DCadsv5, DCesv6, DCasv6 | Confidential computing, data-in-use protection | 2-96 | Hardware-based TEE |

### Compute Optimized (High CPU-to-Memory)

| Family | Series | Use Cases | vCPU Range | Notes |
|--------|--------|-----------|------------|-------|
| **F-family** | Fasv7/Fadsv7, Famsv7, Fsv2 | Batch processing, web servers, analytics, gaming | 2-96 | High CPU performance per core |
| **FX-family** | FX, FXmsv2, FXmdsv2 | EDA, large relational DBs, in-memory analytics | 12-176 | High memory + high CPU |

### Memory Optimized (High Memory-to-CPU)

| Family | Series | Use Cases | vCPU Range | Memory |
|--------|--------|-----------|------------|--------|
| **E-family** | Esv7/Edsv7, Easv7/Eadsv7, Esv6/Edsv6, Ev5/Esv5 | Relational DBs, large caches, in-memory analytics | 2-128 | 8 GiB/vCPU |
| **Eb-family** | Ebsv6/Ebdsv6, Ebdsv5/Ebsv5 | High remote storage performance workloads | 2-128 | E-family + high storage IOPS |
| **EC-family** | ECasv6/ECadsv6, ECasv5/ECadsv5 | Confidential + memory-intensive workloads | 2-96 | E-family + confidential |
| **M-family** | Mbsv3/Mbdsv3, Msv3/Mdsv3, Msv2/Mdsv2 | Extremely large DBs (SAP HANA), massive memory | 2-192 | Up to 4 TB RAM |

### Storage Optimized (High Disk Throughput & IO)

| Family | Series | Use Cases | vCPU Range | Notes |
|--------|--------|-----------|------------|-------|
| **L-family** | Lsv4, Lasv4, Laosv4, Lsv3, Lasv3 | Big Data, NoSQL (Cassandra, MongoDB), data warehousing, large transactional DBs | 8-96 | High local NVMe storage |

### GPU Accelerated

| Family | Series | Use Cases | GPU Type |
|--------|--------|-----------|----------|
| **NC-family** | NC_RTXPRO6000BSE_v6, NCads_H100_v5, NCasT4_v3, NC_A100_v4 | ML training, inference, HPC, rendering | NVIDIA T4, A100, H100, RTX |
| **ND-family** | ND_GB300_v6, ND_GB200_v6, ND_MI300X_v5, ND_H200_v5, ND_H100_v5, ND_A100_v4 | Large-scale deep learning, HPC | NVIDIA H100/H200/GB200, AMD MI300X |
| **NG-family** | NGads_V620 | Virtual desktop (VDI), cloud gaming | AMD Radeon PRO V620 |
| **NV-family** | NVv4, NVadsA10_v5, NVads_V710_v5 | VDI, video encoding, rendering, single-precision | NVIDIA A10, various |

### FPGA Accelerated

| Family | Series | Use Cases | Notes |
|--------|--------|-----------|-------|
| **NP-family** | NP | ML inference, video transcoding, DB search & analytics | Intel FPGA |

### High Performance Compute (HPC)

| Family | Series | Use Cases | Notes |
|--------|--------|-----------|-------|
| **HB-family** | HBv2, HBv3, HBv4, HBv5 | Fluid dynamics, weather modeling | High memory bandwidth, InfiniBand |
| **HC-family** | HC (retiring May 2027) | Finite element analysis, molecular dynamics | Dense compute, InfiniBand |
| **HX-family** | HX | EDA, large memory HPC | Very high memory capacity |

---

## B-Series Detail (Burstable - Most Cost-Effective)

### Bv1 (Previous Gen - Being Retired in Many Regions)

| Size | vCPU | RAM (GiB) | Temp Storage (GiB) | Base CPU % | Credits/hr | Max Burst |
|------|------|-----------|--------------------:|------------|-----------|-----------|
| B1ls | 1 | 0.5 | 4 | 5% | 3 | 100% |
| **B1s** | 1 | 1 | 4 | 10% | 6 | 100% |
| B1ms | 1 | 2 | 4 | 20% | 12 | 100% |
| B2s | 2 | 4 | 8 | 40% | 24 | 200% |
| B2ms | 2 | 8 | 16 | 60% | 36 | 200% |
| B4ms | 4 | 16 | 32 | 90% | 54 | 400% |
| B8ms | 8 | 32 | 64 | 135% | 81 | 800% |
| B12ms | 12 | 48 | 96 | 202% | 121 | 1200% |
| B16ms | 16 | 64 | 128 | 270% | 162 | 1600% |
| B20ms | 20 | 80 | 160 | 337% | 202 | 2000% |

### Bsv2 (Current Gen - Intel)

| Size | vCPU | RAM (GiB) | Temp Storage | Max IOPS | Max Bandwidth (Mbps) |
|------|------|-----------|-------------|----------|---------------------|
| B2s_v2 | 2 | 8 | None | 3750 | 750 |
| B4s_v2 | 4 | 16 | None | 6400 | 1000 |
| B8s_v2 | 8 | 32 | None | 6400 | 1000 |
| B16s_v2 | 16 | 64 | None | 6400 | 2000 |
| B32s_v2 | 32 | 128 | None | 6400 | 5000 |

### Bpsv2 (Current Gen - ARM/Ampere)

| Size | vCPU | RAM (GiB) | Temp Storage | Notes |
|------|------|-----------|-------------|-------|
| B2pts_v2 | 2 | 1 | None | Tiny memory, ARM-based |
| B2pls_v2 | 2 | 4 | None | Low memory, ARM-based |
| B2ps_v2 | 2 | 8 | None | ARM-based |
| B4ps_v2 | 4 | 16 | None | ARM-based |
| B8ps_v2 | 8 | 32 | None | ARM-based |
| B16ps_v2 | 16 | 64 | None | ARM-based |

### Basv2 (Current Gen - AMD)

| Size | vCPU | RAM (GiB) | Temp Storage | Notes |
|------|------|-----------|-------------|-------|
| B2ats_v2 | 2 | 1 | None | Tiny memory, AMD-based |
| B2als_v2 | 2 | 4 | None | Low memory, AMD-based |
| B2as_v2 | 2 | 8 | None | AMD-based |
| B4as_v2 | 4 | 16 | None | AMD-based |
| B8as_v2 | 8 | 32 | None | AMD-based |
| B16as_v2 | 16 | 64 | None | AMD-based |
| B32as_v2 | 32 | 128 | None | AMD-based |

---

## Azure Free Tier

### Overview

| Component | Details |
|-----------|---------|
| **Credit** | $200 for first 30 days (any service) |
| **Duration** | 12 months of free services after signup |
| **Always Free** | 55+ services with permanent free tiers |

### Free Virtual Machines (12 Months)

| VM Size | Type | vCPU | RAM | Free Hours/Month | OS |
|---------|------|------|-----|-----------------|-----|
| **B1s** | Intel (Bv1) | 1 | 1 GiB | 750 | Linux or Windows |
| **B2pts_v2** | ARM (Ampere) | 2 | 1 GiB | 750 | Linux or Windows |
| **B2ats_v2** | AMD | 2 | 1 GiB | 750 | Linux or Windows |

> Total: up to 1,500 free VM hours/month (750 per OS type)

### Free Managed Disks (12 Months)

| Type | Size | Qty | Notes |
|------|------|-----|-------|
| P6 SSD | 64 GiB | 2 | Premium SSD |
| Snapshot | 1 GB | 1 | - |
| I/O Operations | 2 million | - | Monthly |

### Other Free Services (12 Months)

| Service | Free Tier Amount |
|---------|-----------------|
| **Blob Storage** | 5 GB LRS hot block, 20K read / 10K write ops |
| **File Storage** | 100 GB LRS (transaction optimized, hot, cool), 2M operations |
| **MySQL Flexible Server** | 750 hrs Burstable B1MS, 32 GB storage, 32 GB backup |
| **PostgreSQL Flexible Server** | 750 hrs Burstable B1MS, 32 GB storage, 32 GB backup |
| **SQL Managed Instance** | 720 vCore hours (4 or 8 vCore GP), 64 GB data + 64 GB backup |
| **Cosmos DB** | 400 RU/s, 25 GB storage |
| **Bandwidth** | 15 GB outbound data transfer |
| **Load Balancer** | 750 hrs Standard, 15 GB data processing, 5 rules |
| **VPN Gateway** | 750 hrs VpnGw1 |
| **Key Vault** | 10,000 transactions (RSA 2048-bit) |
| **Container Registry** | 1 Standard tier, 100 GB storage, 10 webhooks |
| **Service Bus** | 750 hrs Standard tier, 13M operations |
| **App Configuration** | 1,000 requests/day, 10 MB storage |
| **AI Document Intelligence** | 500 pages S0 tier |
| **Computer Vision** | 5,000 transactions (S1/S2/S3) |
| **Face API** | 30,000 transactions S0 tier |

### Always-Free Services (Permanent)

| Service | Free Amount |
|---------|-------------|
| **Azure SQL Database** | 100,000 vCore seconds, 32 GB storage (up to 10 DBs) |
| **Cosmos DB** | 1,000 RU/s, 25 GB storage (lifetime free tier) |
| **Azure Functions** | 1 million requests/month |
| **Azure Static Web Apps** | 100 GB bandwidth, 2 custom domains, 0.5 GB storage |
| **Azure Container Apps** | 180,000 vCPU sec, 360,000 GiB sec, 2M requests |
| **Azure Kubernetes Service** | Cluster management free (pay only for nodes) |
| **Azure DevOps** | 5 users, unlimited private Git repos |
| **Azure Monitor** | Free tier amounts per feature |
| **Event Grid** | 100,000 operations/month |
| **API Management** | 1 million calls/month (Consumption tier) |
| **Notification Hubs** | 1 million push notifications |
| **Azure Active Directory** | 50,000 stored objects, SSO to all cloud apps |
| **Azure AD B2C** | 50,000 MAU |
| **IoT Hub** | 8,000 messages/day (Free edition) |
| **Advisor** | Unlimited recommendations |
| **Logic Apps** | 4,000 built-in actions (Consumption plan) |
| **Azure Migrate** | Free |
| **Cost Management** | Free |
| **Virtual Network** | 50 virtual networks |
| **Cognitive Services - Speech** | 5 audio hours/month |
| **Cognitive Services - Translator** | 2 million characters/month |
| **Cognitive Services - Language** | 5,000 text records |
| **Azure Maps** | 1,000-5,000 transactions |
| **SignalR** | 20 concurrent connections, 20,000 messages/day |
| **Web PubSub** | 20 concurrent connections, 20,000 messages |

---

## Azure Regions (All Available)

### Americas

| Region Name | Programmatic Name | Location | Paired Region | AZ Support |
|-------------|-------------------|----------|---------------|:----------:|
| East US | `eastus` | Virginia, US | West US | Yes |
| East US 2 | `eastus2` | Virginia, US | Central US | Yes |
| Central US | `centralus` | Iowa, US | East US 2 | Yes |
| North Central US | `northcentralus` | Illinois, US | South Central US | No |
| South Central US | `southcentralus` | Texas, US | North Central US | Yes |
| West Central US | `westcentralus` | Wyoming, US | West US 2 | No |
| West US | `westus` | California, US | East US | No |
| West US 2 | `westus2` | Washington, US | West Central US | Yes |
| West US 3 | `westus3` | Phoenix, US | East US | Yes |
| Canada Central | `canadacentral` | Toronto, Canada | Canada East | Yes |
| Canada East | `canadaeast` | Quebec, Canada | Canada Central | No |
| Brazil South | `brazilsouth` | Sao Paulo, Brazil | South Central US | Yes |
| Brazil Southeast | `brazilsoutheast` | Rio, Brazil | Brazil South | No (restricted) |
| Mexico Central | `mexicocentral` | Querétaro, Mexico | N/A | Yes |
| Chile Central | `chilecentral` | Santiago, Chile | N/A | Yes |

### Europe

| Region Name | Programmatic Name | Location | Paired Region | AZ Support |
|-------------|-------------------|----------|---------------|:----------:|
| North Europe | `northeurope` | Ireland | West Europe | Yes |
| West Europe | `westeurope` | Netherlands | North Europe | Yes |
| UK South | `uksouth` | London, UK | UK West | Yes |
| UK West | `ukwest` | Cardiff, UK | UK South | No |
| France Central | `francecentral` | Paris, France | France South | Yes |
| France South | `francesouth` | Marseille, France | France Central | No (restricted) |
| Germany West Central | `germanywestcentral` | Frankfurt, Germany | Germany North | Yes |
| Germany North | `germanynorth` | Berlin, Germany | Germany West Central | No (restricted) |
| Switzerland North | `switzerlandnorth` | Zurich, Switzerland | Switzerland West | Yes |
| Switzerland West | `switzerlandwest` | Geneva, Switzerland | Switzerland North | No (restricted) |
| Norway East | `norwayeast` | Norway | Norway West | Yes |
| Norway West | `norwaywest` | Norway | Norway East | No (restricted) |
| Sweden Central | `swedencentral` | Gävle, Sweden | Sweden South | Yes |
| Poland Central | `polandcentral` | Warsaw, Poland | N/A | Yes |
| Italy North | `italynorth` | Milan, Italy | N/A | Yes |
| Spain Central | `spaincentral` | Madrid, Spain | N/A | Yes |
| Austria East | `austriaeast` | Vienna, Austria | N/A | Yes |
| Belgium Central | `belgiumcentral` | Brussels, Belgium | N/A | Yes |
| Denmark East | `denmarkeast` | Copenhagen, Denmark | N/A | Yes |
| Finland Central | `finlandcentral` | Coming soon | N/A | Yes |
| Greece Central | `greececentral` | Coming soon | N/A | Yes |

### Middle East & Africa

| Region Name | Programmatic Name | Location | Paired Region | AZ Support |
|-------------|-------------------|----------|---------------|:----------:|
| UAE North | `uaenorth` | Dubai, UAE | UAE Central | Yes |
| UAE Central | `uaecentral` | Abu Dhabi, UAE | UAE North | No (restricted) |
| Qatar Central | `qatarcentral` | Doha, Qatar | N/A | Yes |
| Israel Central | `israelcentral` | Israel | N/A | Yes |
| South Africa North | `southafricanorth` | Johannesburg | South Africa West | Yes |
| South Africa West | `southafricawest` | Cape Town | South Africa North | No (restricted) |
| Saudi Arabia (coming) | - | - | N/A | - |

### Asia Pacific

| Region Name | Programmatic Name | Location | Paired Region | AZ Support |
|-------------|-------------------|----------|---------------|:----------:|
| East Asia | `eastasia` | Hong Kong | Southeast Asia | Yes |
| Southeast Asia | `southeastasia` | Singapore | East Asia | Yes |
| Japan East | `japaneast` | Tokyo/Saitama, Japan | Japan West | Yes |
| Japan West | `japanwest` | Osaka, Japan | Japan East | No |
| Korea Central | `koreacentral` | Seoul, Korea | Korea South | Yes |
| Korea South | `koreasouth` | Busan, Korea | Korea Central | No |
| Central India | `centralindia` | Pune, India | South India | Yes |
| South India | `southindia` | Chennai, India | Central India | No |
| West India | `westindia` | Mumbai, India | South India | No |
| India South Central | `indiasouthcentral` | Hyderabad, India | Central India | Yes |
| Australia East | `australiaeast` | New South Wales | Australia Southeast | Yes |
| Australia Southeast | `australiasoutheast` | Victoria | Australia East | No |
| Australia Central | `australiacentral` | Canberra | Australia Central 2 | Yes |
| Australia Central 2 | `australiacentral2` | Canberra | Australia Central | No (restricted) |
| Indonesia Central | `indonesiacentral` | Jakarta, Indonesia | N/A | Yes |
| Malaysia West | `malaysiawest` | Kuala Lumpur | N/A | Yes |
| New Zealand North | `newzealandnorth` | Auckland, NZ | N/A | Yes |
| Taiwan (coming) | - | - | N/A | - |

### Sovereign Clouds (Separate from Public Azure)

| Cloud | Regions | Notes |
|-------|---------|-------|
| **Azure Government** | US Gov Virginia, US Gov Iowa, US Gov Arizona, US Gov Texas, US DoD East, US DoD Central | US government only |
| **Azure China** | China East, China East 2, China East 3, China North, China North 2, China North 3 | Operated by 21Vianet |

### Region Selection Tips

- **Lowest latency**: Choose the region closest to your users
- **Paired regions**: Use for disaster recovery (data stays in same geography)
- **Availability Zones (AZ)**: 3 independent zones per AZ-enabled region for HA
- **Pricing varies by region**: US East/West tend to be cheapest; Brazil/Australia premium
- **Free tier VMs**: Available in any region where B-series is offered (check with `az vm list-skus`)
- **Restricted regions**: Require access request via Azure support

```bash
# List all regions
az account list-locations --output table

# List regions with AZ support
az account list-locations --query "[?availabilityZoneMappings != null].{Name:name, DisplayName:displayName}" --output table

# Check VM availability in a specific region
az vm list-skus --location westeurope --resource-type virtualMachines --output table
```

---

## Quick Comparison: Which Series to Choose?

| Workload | Recommended Series | Why |
|----------|-------------------|-----|
| Dev/test, low traffic | **B-series** | Cheapest, burstable credits |
| General web apps, enterprise | **D-series** | Balanced price/performance |
| CPU-intensive batch jobs | **F-series** | Best CPU per dollar |
| Databases, large caches | **E-series** | 8 GiB RAM per vCPU |
| SAP HANA, massive memory | **M-series** | Up to 4 TB RAM |
| Big Data, NoSQL | **L-series** | High local NVMe throughput |
| ML Training, AI | **NC/ND-series** | NVIDIA GPUs (T4 to H200) |
| Virtual desktops, rendering | **NV/NG-series** | GPU for graphics |
| HPC (weather, CFD) | **HB-series** | High bandwidth, InfiniBand |
| Confidential computing | **DC/EC-series** | Hardware-based encryption |
| ARM workloads (cost saving) | **Dps/Eps/Bps** | ~20% cheaper, Ampere CPUs |

---

## Useful CLI Commands

```bash
# List all VM sizes available in a region
az vm list-skus --location eastus --resource-type virtualMachines --output table

# List only specific series
az vm list-skus --location eastus --resource-type virtualMachines --size Standard_B --output table

# List sizes with pricing (requires az extension)
az vm list-sizes --location eastus --output table

# Check free tier eligibility
az vm list-skus --location eastus --size Standard_B1s --output table

# Create a free-tier eligible VM (Linux)
az vm create \
  --resource-group myRG \
  --name myFreeVM \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys

# Create ARM-based free-tier VM
az vm create \
  --resource-group myRG \
  --name myArmVM \
  --image Canonical:ubuntu-24_04-lts:server-arm64:latest \
  --size Standard_B2pts_v2 \
  --admin-username azureuser \
  --generate-ssh-keys
```

---

## Tips to Stay on Free Tier

1. **Use B1s, B2pts_v2, or B2ats_v2** — only these VM sizes qualify
2. **Monitor hours** — 750 hrs/month per VM type (enough for 1 VM running 24/7)
3. **Use P6 disks** — 64 GiB SSD is the free disk tier
4. **Set billing alerts** — configure budget alerts at $0.01 to catch charges early
5. **Use the Free Services portal page** — it pre-selects free-eligible configurations
6. **One VM per type** — running 2x B1s for 400 hrs each = 800 hrs (exceeds free limit)
7. **Check region availability** — B1s is being retired in some regions; prefer B2ats_v2 or B2pts_v2
