# QuantLib Financial Instruments Documentation

Professional documentation for pricing financial instruments with QuantLib.

## Supported Instruments

### Equity (8 instruments)
- Equity Swap, Index Swap
- American/European Options
- Warrants, Futures

### Fixed Income (3 instruments)
- Treasury Bonds
- Callable Bonds
- Floating Rate Notes

### Credit (3 instruments)
- Credit Default Swaps
- CDS Indices
- CDX/iTraxx Options

### Rates & FX (2 instruments)
- Interest Rate Swaps
- Currency Forwards

### Cash Flow (4 types)
- Non-Performing
- Performing (Fixed/Floating/Supplied)

## Quick Start

```bash
# Install
cd ~/SoftwareProjects/quantlib-instruments-docs
source venv/bin/activate
pip install -r requirements.txt

# Serve documentation
mkdocs serve
```

Open http://localhost:8000

## Build Documentation

```bash
mkdocs build
```

## Features

- Complete instrument coverage
- QuantLib code examples
- Interactive Jupyter notebooks
- Mermaid diagrams
- Professional Material theme

## Documentation Structure

- `docs/instruments/` - All instrument documentation
- `docs/examples/` - Jupyter notebooks
- `docs/guides/` - User guides
- `mkdocs.yml` - Configuration

---

Created for demonstration purposes
