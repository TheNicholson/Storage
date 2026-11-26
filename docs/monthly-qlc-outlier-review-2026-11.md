# Monthly QLC outlier review — November 2026

Review of public QLC NVMe classification in [vmware/vcf-readiness](https://github.com/vmware/vcf-readiness) at commit `b278505964004e535d9f669790d91745431359b4` (20 November 2026, “Prepare sanitized VCF Readiness 9.14.0”). This repository stores the review. The code under review is the public VCF Readiness tree.

## What was reviewed

`detect_qlc_nvme()` in `vcf_hci/hcl/cross_reference.py` returns true when the uppercased model contains the substring `QLC`, or when the model starts with any entry in `QLC_NVME_PREFIXES` (`vcf_hci/constants.py`). The collector sets `is_qlc` from that function for direct-attached NVMe (`vcf_hci/collector/drive_topology.py`). Two later uses matter:

- If a drive has no vSAN HCL match, `evaluate_drive_hcl_tier()` labels it “QLC NVMe — Not vSAN Certified” instead of the generic unverified badge.
- `namespace_candidate_source()` in `vcf_hci/hcl/memory_tiering.py` returns no TLC 1 DWPD evidence when `detect_qlc_nvme()` is true. That check runs before the `RI_SOURCES` patterns, so a prefix hit suppresses a later TLC product brief for the same model.

An HCL match still wins the vSAN badge. A false QLC prefix does not relabel a drive that already matched ESA or OSA. It does block the memory-tiering TLC candidate path, and it supplies the QLC badge when the drive is off the HCL.

Vendor NAND type below is taken from the cited product specification or datasheet, checked on 26 November 2026.

## Outliers

These prefixes classify TLC, SCM, or the wrong product as QLC. Ordered by how many common enterprise models they sweep in.

### `MTFDL` — Micron Gen5, not QLC

The comment says every `MTFDL*` part is a 6600 ION QLC drive. Micron’s own part charts use `MTFDL` for the PCIe Gen5 interface on more than one NAND type:

| Family | Example | NAND in the vendor spec |
| --- | --- | --- |
| 9550 PRO / MAX | `MTFDLAL15T3THA`, `MTFDLBQ3T8THA` | G8 TLC. Product family `HA` / `HB`. |
| 6550 ION | `MTFDLAL30T7THL` | TLC. The March 2026 transition guide calls both 6500 ION and 6550 ION TLC. |
| 6600 ION | `MTFDLAL30T7QHF` | G9 QLC. The chart marks the NAND character as `Q` and the family as `HF`. |

Sources: [9550 technical product specification](https://assets.micron.com/adobe/assets/urn:aaid:aem:8c1e2b9b-d81a-45f0-9a9f-8edcd6c23ec9/renditions/original/as/9550-nvme-ssd-tech-prod-spec.pdf), [6600 ION technical product specification](https://www.micron.com/content/dam/micron/global/public/products/storage/ssds/data-center/6600/6600-ion-ssd-technical-product-specification.pdf), [6500 ION transition guide](https://www.edgeelectronics.com/resources/micron-6500-ion-eol-transition-guide/) (27 March 2026).

A 9550 or 6550 model string therefore sets `is_qlc`. The 6600 discriminator published by Micron is the `QHF` family code, not the `MTFDL` interface prefix.

### `MTFDKCC` and `MTFDKBN` — 6500 ION is TLC, and `MTFDKCC` is also 9400

The comment calls the 6500 ION a QLC drive and cites `MTFDKCC30T7TGR` and `MTFDKBN` (E1.L). Micron’s 6500 ION specification says “Micron 3D TLC NAND Flash”. The same `MTFDKCC` form-factor code is the 9400 PRO (`GH`) and 9400 MAX (`GJ`), also TLC. Examples: `MTFDKCC30T7TGH` is 9400 PRO; `MTFDKCC30T7TGR` is 6500 ION. The suffix distinguishes them. The shared prefix does not.

Sources: [6500 ION technical product specification](https://assets.micron.com/adobe/assets/urn:aaid:aem:55527cf3-550d-4873-b99b-4f02404ac36d/renditions/original/as/6500-ion-nvme-ssd-tech-prod-spec.pdf), [9400 technical product specification](https://assets.micron.com/adobe/assets/urn:aaid:aem:83accf2c-00af-4223-ab00-c40fb224ce6c/renditions/original/as/9400-nvme-ssd-tech-prod-spec.pdf).

The comment’s claim that `MTFDKBA` (described there as 9400 PRO) is safely excluded does not hold. The 9400 U.2/U.3 chart uses `CC`, which is the prefix in the QLC list.

### `KCD8` — CD8-R and CD8P are TLC

The comment says `KCD8` is “CD8P-V (QLC)” and is broader than `KCD8P` so future CD8 variants are caught. Kioxia’s current briefs say otherwise:

- CD8-R (`KCD81RUG`, `KCD8XRUG`, `KCD8DRUG`) is BiCS FLASH TLC, 1 DWPD.
- CD8P-V (`KCD81PUG`, `KCD8XPUG`, `KCD8DPUG`) is BiCS FLASH generation 5 TLC, 3 DWPD.

Sources: [CD8-R product brief](https://www.kioxia.com/content/dam/kioxia/shared/business/ssd/data-center-ssd/asset/productbrief/dSSD-CD8-R-U2-product-brief.pdf), [CD8P-V product brief](https://www.kioxia.com/content/dam/kioxia/shared/business/ssd/data-center-ssd/asset/productbrief/dSSD-CD8P-V-U2-product-brief.pdf).

This collides with code in the same tree. `RI_SOURCES` in `memory_tiering.py` treats `CD8-R` / `KCD8[1XD]RUG…` as reviewed TLC 1 DWPD evidence. `namespace_candidate_source()` never reaches that pattern, because `detect_qlc_nvme("KCD81RUG15T3")` is already true.

### `WUS4CB` — Ultrastar DC SN640 TLC, not SN860 QLC

The comment says “WD Ultrastar DC SN860 (QLC NVMe)”. Western Digital’s SN640 datasheet lists `WUS4CB080D7P3E`, `WUS4CB016D7P3E`, `WUS4CB032D7P3E`, and `WUS4CB064D7P3E` as the 2 DW/D SKUs, with BiCS4 3D TLC NAND. The 0.8 DW/D siblings are `WUS4BB*`, which this prefix does not match. No SN860 model number in that datasheet uses `WUS4CB`.

Source: [Ultrastar DC SN640 data sheet](https://documents.sandisk.com/content/dam/asset-library/en_us/assets/public/western-digital/product/data-center-drives/ultrastar-nvme-series/data-sheet-ultrastar-dc-sn640.pdf).

### `KCF8` and `KFF8` — not the published FL6 parts, and FL6 is not QLC

The comment says these are FL6 EDSFF QLC. Kioxia’s FL6 brief lists `KFL61HUL`, `KFL6XHUL`, `KFL6DHUL`, and `KFL6FHUL`: 2.5-inch U.3, XL-FLASH 1-bit/cell SLC, 60 DWPD. `KFL6` is not in `QLC_NVME_PREFIXES`. This pass did not find a Kioxia QLC model that starts with `KCF8` or `KFF8`.

Source: [FL6 product brief](https://americas.kioxia.com/content/dam/kioxia/shared/business/ssd/enterprise-ssd/asset/productbrief/eSSD-FL6-product-brief.pdf).

### `CM5` — three characters, and CM5 is TLC

The comment says “Dell Express Flash CM5 NVMe (OEM-branded QLC)”. Kioxia CM5 (the enterprise family Dell and HPE have shipped) is 64-layer BiCS FLASH TLC. Published part numbers start with `KCM5` (`KCM51RUG15T3` and similar), so `startswith("CM5")` misses those strings. It does match any model field that literally begins with `CM5`, which is the TLC family name.

Source: [Kioxia CM5-R KCM51RUG15T3](https://serverorbit.com/kioxia-kcm51rug15t3-cm5-r-series-15-36tb-nvme-ri-ssd/) and Kioxia’s CM5 versus CD5 comparison, both describing 64-layer 3D TLC.

### `STJN` and `SHGP` — comments do not match the public model lists

- `STJN` is commented as Seagate Nytro 5550H QLC, EDSFF E1.S. Seagate’s Nytro 5050 manual (September 2024) lists 5550H as 3D eTLC, 3 DWPD, U.2/U.3, with model numbers `XP12800LE70005` and the rest of the `XP*` set. No `STJN` model appears there, and the manual does not describe an E1.S 5550H.
- `SHGP` is commented as SK hynix PE8010 / PE8011 QLC. SK hynix describes PE8010 and PE8030 as 96-layer / V5 TLC, 1 DWPD and 3 DWPD. This pass did not find a public PE8010 part number beginning `SHGP`, so the prefix may be an unpublished OEM string or a stale alias. The product named in the comment is TLC either way.

Sources: [Nytro 5050 product manual](https://www.seagate.com/content/dam/seagate/assets/support/enterprise-storage/solid-state-drives/Nytro-5050-NVMe-SSD-Series/_shared/file/Seagat_Nytro_5050_NVMe_SSD-202252400D.pdf), [SK hynix PE8000 series](https://product.skhynix.com/products/ssd/essd/pe8000.go).

## Prefixes that were not false positives in this pass

- `SBFPF` is commented as Solidigm D5-P5430 only. `SBFPF2BU153T` is P5430 QLC. `SBFPF2BV153T` is D5-P5336, also QLC, so the prefix is wider than the comment and still QLC for those two series. D7-P5520 and D7-P5620 TLC parts in Solidigm’s September 2024 firmware PCN use `SSDPF*`, not `SBFPF`.
- `SSDPE2NV` matches the older Intel D5-P4326 QLC numbering pattern. This pass did not re-audit every capacity.
- `RLC9` matches the Kioxia LC9 QLC family named in the comment. No TLC collision showed up in this pass.
- `MZWMO` and `MZ3MO` are narrow, and the comment already refuses a blanket `MZWL` prefix because `MZWLJ` is PM9A1 TLC. A Samsung BM1743 brief was not re-opened this month.

## Related alias outlier

`_MODEL_FAMILY_ALIASES` in `cross_reference.py` groups `MZWMO` and `MZ3MO` with `PM1743` and `BM1743` in one alias. PM1743 and BM1743 are different NAND generations (TLC performance versus QLC capacity). Fuzzy HCL scoring can award an alias hit to either name. A BM1743 model can therefore score against a PM1743 HCL row when the capacity token also lines up.

## Coverage gaps, separate from the false positives

- Samsung PM9C3 is intentionally absent. The comment says `MZWL` is not a safe prefix and that PM9C3 waits on a unique confirmed prefix or the literal `QLC` token.
- Micron 6600 ION QLC is only reached today by the over-broad `MTFDL` prefix. A match on the `QHF` family code would keep 6600 without taking 9550 and 6550 with it.
- Intel D5-P5316 QLC parts use `SSDPF2NV…`. The list has `SSDPE2NV` (P4326) and does not have `SSDPF2NV`. Solidigm-branded D5-P5336 `SBFPF2BV` is already covered by `SBFPF`.

## What changed since the October releases

`vcf_hci/constants.py` was last touched in the 9.14.0 commit on 20 November 2026. The October release commits (9.8.2 through 9.12.1) also modified that file. This review is against the tree those commits produced, not a diff of one prefix. The false `KCD8` versus `RI_SOURCES` contradiction is present together in 9.14.0.

## Suggested correction, not applied here

This review does not change vmware/vcf-readiness.

1. Drop `MTFDKCC`, `MTFDKBN`, `KCD8`, `WUS4CB`, `KCF8`, `KFF8`, and `CM5` from `QLC_NVME_PREFIXES`.
2. Replace `MTFDL` with a 6600 ION marker such as `QHF`, and add a regression that `MTFDLAL15T3THA` (9550 PRO) and `MTFDLAL30T7THL` (6550 ION) stay false.
3. Add a regression that `KCD81RUG15T3` stays false and still resolves through the CD8-R entry in `RI_SOURCES`.
4. Split the `PM1743` / `BM1743` alias so an `MZWMO` model cannot score as PM1743.
5. Re-check `STJN` and `SHGP` against a vendor part list before putting either prefix back. Leave `SBFPF`, and extend its comment to include D5-P5336 (`SBFPF2BV`).
