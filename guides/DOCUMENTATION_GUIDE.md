# Documentation Organization Guide

## 📁 Documentation Structure

All documentation is organized in the repository root:

```
.
├── README.md                          # Repository README
├── index.qmd                          # Documentation home page
├── guides/                           # Developer guides
│   ├── QUICK_START.md                # Fast-track guide (~20 min)
│   ├── ADDING_NEW_EXERCISES.md       # Complete reference
│   ├── INTERACTIVE_DISCOVERY.md      # Discovery tutorial
│   ├── DOCUMENTATION_GUIDE.md        # This file
│   └── BUILD_SITE.md                 # Building documentation
├── examples/                         # Real-world examples
│   └── BCM_WELCOME_MEALS.md          # Complete example walkthrough
└── development/                      # Development notes
```

## 🎯 Which Document to Use?

### I want to add an exercise quickly
→ **[QUICK_START.md](QUICK_START.md)**

- Copy-paste template
- Essential steps only
- **Time**: ~20 minutes

### I want to learn the discovery process
→ **[INTERACTIVE_DISCOVERY.md](INTERACTIVE_DISCOVERY.md)**

- Step-by-step tutorial
- Exploration techniques
- Real examples
- **Time**: ~30 minutes

### I need complete reference information
→ **[ADDING_NEW_EXERCISES.md](ADDING_NEW_EXERCISES.md)**

- All validation rules
- All UI components
- Complete troubleshooting
- Best practices
- **Time**: Reference as needed

### I want to see a real example
→ **[../examples/BCM_WELCOME_MEALS.md](../examples/BCM_WELCOME_MEALS.md)**

- Actual implementation
- Design decisions
- Testing checklist
- **Time**: ~15 minutes

### I want to understand the app architecture
→ **[../README.md](../README.md)**

- System overview
- Component descriptions
- Data flow diagrams
- **Time**: ~20 minutes

## 🔧 Tools (Not Documentation)

These tools live in the **project root** (not in docs/):

### `tools/discover_surveys.r`
Interactive survey discovery tool:
```r
source("tools/discover_surveys.r")
search_surveys("keyword")
inspect_survey(SURVEY_ID)
```

### `explore_bcm.r`
Example exploration script showing how to analyze survey data.

## 🚀 Quick Navigation

From the **repository root**:

```bash
# Repository README
cat README.md

# Documentation home page
cat index.qmd

# Quick start guide
cat guides/QUICK_START.md

# Discovery tutorial
cat guides/INTERACTIVE_DISCOVERY.md

# Complete reference
cat guides/ADDING_NEW_EXERCISES.md

# Real example
cat examples/BCM_WELCOME_MEALS.md
```

## 📊 Documentation Hierarchy

```
README.md (Repository README)
    ↓
index.qmd (Documentation Home Page)
    ↓
    ├─→ guides/QUICK_START.md (Fast path)
    ├─→ guides/INTERACTIVE_DISCOVERY.md (Learning)
    ├─→ guides/ADDING_NEW_EXERCISES.md (Reference)
    └─→ examples/BCM_WELCOME_MEALS.md (Example)
```

## 🎓 Recommended Learning Path

### Beginner (1 hour)

1. Read root `README.md` (5 min)
2. Run discovery tool with example (10 min)
3. Read `guides/QUICK_START.md` (15 min)
4. Study `examples/BCM_WELCOME_MEALS.md` (15 min)
5. Try creating a simple exercise (15 min)

### Intermediate (2 hours)

1. Complete beginner path
2. Read `guides/INTERACTIVE_DISCOVERY.md` (30 min)
3. Review all validation rules in code (30 min)
4. Create exercise with custom validations (30 min)

### Advanced (3+ hours)

1. Complete intermediate path
2. Read `guides/ADDING_NEW_EXERCISES.md` completely
3. Study registry system code
4. Create custom validation rules
5. Build custom visualization components

## 🔍 Finding Information

### By Topic

| Topic | Document | Section |
|-------|----------|---------|
| Survey ID discovery | `guides/INTERACTIVE_DISCOVERY.md` | Method 1 |
| Exercise structure | `guides/ADDING_NEW_EXERCISES.md` | Exercise Structure Reference |
| Validation rules | `guides/ADDING_NEW_EXERCISES.md` | Validation Rules Reference |
| UI components | `guides/ADDING_NEW_EXERCISES.md` | Visualization Reference |
| Troubleshooting | `guides/ADDING_NEW_EXERCISES.md` | Troubleshooting |
| Real example | `examples/BCM_WELCOME_MEALS.md` | All sections |
| Architecture | `index.qmd` | Application Architecture |

### By Task

| Task | Start Here |
|------|-----------|
| Add new exercise | `guides/QUICK_START.md` |
| Find survey ID | Run `tools/discover_surveys.r` |
| Understand discovery | `guides/INTERACTIVE_DISCOVERY.md` |
| Learn validation rules | `guides/ADDING_NEW_EXERCISES.md` → Validation Rules Reference |
| Create custom dashboard | `guides/ADDING_NEW_EXERCISES.md` → Visualization Reference |
| Debug issues | `guides/ADDING_NEW_EXERCISES.md` → Troubleshooting |

## 📝 Document Sizes

| Document | Size | Read Time |
|----------|------|-----------|
| Repository README | 3 KB | 3 min |
| Documentation Home (index.qmd) | 7 KB | 5 min |
| QUICK_START | 5 KB | 5 min |
| INTERACTIVE_DISCOVERY | 19 KB | 20 min |
| ADDING_NEW_EXERCISES | 52 KB | 60 min (reference) |
| BCM_WELCOME_MEALS | 10 KB | 10 min |

## 🔄 Documentation Updates

### When to Update Documentation

**Add to examples/** when:

- You create a notable new exercise
- You implement a new pattern
- You solve a complex problem

**Update guides/** when:

- You add new validation rules
- You add new UI components
- You change the exercise structure
- You find better ways to do things

**Update README files** when:

- Architecture changes
- New tools are added
- Major features are added

### How to Update

1. Make changes to the appropriate file in `docs/`
2. Update the table of contents if structure changed
3. Update cross-references if links changed
4. Test all code examples
5. Check all links still work

## 🎯 Goals of This Organization

### Clarity

- Each document has a clear purpose
- Easy to find the right document
- No duplicate information

### Accessibility

- Multiple entry points for different needs
- Quick start for fast results
- Deep dives for learning
- Reference for details

### Maintainability

- Organized by purpose
- Clear separation of concerns
- Easy to update specific sections

### Discoverability

- Clear navigation from repository README
- Documentation home page (index.qmd)
- Cross-references between documents

## 🚦 Status Indicators

| Document | Status | Last Updated |
|----------|--------|--------------|
| README.md | ✅ Current | 2025-11-30 |
| index.qmd | ✅ Current | 2025-11-30 |
| QUICK_START.md | ✅ Current | 2025-11-30 |
| INTERACTIVE_DISCOVERY.md | ✅ Current | 2025-11-30 |
| ADDING_NEW_EXERCISES.md | ✅ Current | 2025-11-30 |
| BCM_WELCOME_MEALS.md | ✅ Current | 2025-11-30 |

---

**Questions?** Start with the [Documentation Home](../index.html) or the [Repository README](../README.md).

