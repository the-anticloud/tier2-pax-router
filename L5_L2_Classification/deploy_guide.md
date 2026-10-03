# Deploy Guide — PAX_ROUTER
**Stack:** Python 3.11, PAX 27B (classifier head), asyncio, AIOSS_FORMAT | Air-gap capable

## Prerequisites
Anticloud core stack installed. PAX 27B weights (pax-27b-q4.gguf). AIOSS_FORMAT.

## Install
```bash
pip install anticloud-pax-router
```

## AIOSS Integration
```bash
aioss init --module PAX_ROUTER --output ./pax_router.aioss
```

## Air-Gap
```bash
pip download -r requirements.txt -d ./wheels/
pip install --no-index --find-links ./wheels/ -r requirements.txt
```

## PAX Harness Wiring
```python
from anticloud_pax import PAXHarness
harness = PAXHarness(model_path="./pax-27b-q4.gguf", module="PAX_ROUTER",
                     aioss_chain="./pax_router.aioss",
                     classification="L5_NARROW_L2_GENERAL")
```

## Verification
```bash
aioss verify --chain ./pax_router.aioss --verbose
python -m pax_router.tests.smoke
```
