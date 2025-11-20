# Integration Approaches Comparison

**Comparing Two Strategies for Medweb CSV Integration with RadIMO**

---

## 📋 Context

**Goal:** Integrate medweb CSV schedule data directly into RadIMO to eliminate manual Excel file creation.

**Current Workflow:**
```
medweb.csv → SBZDashboard → 3 Excel files → Manual upload to RadIMO
```

**Desired Workflow:**
```
medweb.csv → [Integration Layer] → RadIMO modality_data
```

---

## 🔄 Two Proposed Approaches

### **Approach A: Excel Generation (Original Plan)**
*From `docs/INTEGRATION_PLAN.md`*

**Flow:**
```
medweb.csv
    ↓
Merge SBZDashboard code into RadIMO
    ↓
Generate unified multi-sheet Excel
├── ct (sheet)
├── mr (sheet)
├── xray (sheet) ← Chir mapped here
└── info (sheet)
    ↓
Parse Excel → working_hours_df
    ↓
RadIMO balancer
```

**Key Features:**
- Preserve SBZDashboard's complex logic (shift calculations, Besonderheiten, role detection)
- Generate Excel as intermediate format
- Multi-sheet structure for organization
- Backward compatible with manual Excel upload
- Can preview/export Excel before applying

**Implementation:**
- Copy SBZDashboard's `build_final_df()`, `generate_ctsbz_excel()` functions
- Add `/schedule-generator` route
- Parse generated Excel with existing logic (minor modifications for multi-sheet)

---

### **Approach B: Direct CSV Ingestion (Team Member's Proposal)** ⭐ RECOMMENDED
*From team member's analysis*

**Flow:**
```
medweb.csv
    ↓
Config-driven mapping rules (medweb_mapping in config.yaml)
    ↓
build_working_hours_from_medweb()
    ↓
Apply worker_skill_roster overrides
    ↓
working_hours_df (directly)
    ↓
RadIMO balancer
```

**Key Features:**
- **Config-Driven:** All activity → modality/skill mappings in config.yaml
- **No Excel:** Directly build working_hours_df from CSV
- **Roster System:** Per-worker skill overrides with -1/0/1 semantics
- **Cleaner Architecture:** Separation of mapping rules from code
- **Extensible:** Add new activities just by updating config
- **Optional Excel Export:** Can still generate Excel for backup/review

**Implementation:**
- Add `medweb_mapping` section to config.yaml
- Add `worker_skill_roster` section to config.yaml
- Create `build_working_hours_from_medweb()` function
- Update `/upload` to accept medweb CSV

---

## 🎯 Detailed Comparison

### **1. Activity Mapping**

| Aspect | Approach A: Excel | Approach B: Config-Driven |
|--------|------------------|---------------------------|
| **Mapping Logic** | Hardcoded in Python (row_relevant, extract_modalitaet_und_schicht) | Config.yaml rules |
| **Adding New Activity** | Modify Python code, redeploy | Edit config.yaml, restart |
| **Maintainability** | Medium (code changes) | High (config changes) |
| **Transparency** | Logic buried in code | Visible in config |

**Example Config (Approach B):**
```yaml
medweb_mapping:
  rules:
    - match: "CT Spätdienst"
      modality: "ct"
      shift: "Spaetdienst"
      base_skills: {Normal: 1, Notfall: 1, Herz: 0, Privat: 0}

    - match: "MR Assistent 1. Monat"
      modality: "mr"
      shift: "Fruehdienst"
      base_skills: {Normal: 1, Notfall: 0, Herz: 0, Privat: 0}  # No Notfall for 1. Monat

    - match: "Chir Assistent"
      modality: "xray"  # Chir → xray per user clarification
      shift: "Fruehdienst"
      base_skills: {Normal: 1, Notfall: 1, Herz: 0, Privat: 0}

    - match: "SBZ: MRT-OA"
      modality: "mr"
      shift: "Fruehdienst"
      base_skills: {Normal: 0, Notfall: 0, Herz: 0, Privat: 1}  # PP role
```

---

### **2. Skill Assignment**

| Aspect | Approach A: Excel | Approach B: Config-Driven |
|--------|------------------|---------------------------|
| **Base Skills** | Calculated in build_final_df() | From medweb_mapping rules |
| **Worker Overrides** | skill_overrides (binary 0/1) in config.yml | worker_skill_roster (-1/0/1) in config.yaml |
| **Msk/Chest Assignment** | Hardcoded ID lists (msk_id, chest_id) | worker_skill_roster or base_skills |
| **Cardiac Label** | Besonderheiten keyword matching | Can use label matching OR roster |

**Example Worker Roster (Approach B):**
```yaml
worker_skill_roster:
  # Per-worker skill overrides
  KRUE:
    default:  # Applies to all modalities unless overridden
      Normal: 1
      Notfall: 1
      Herz: 0
      Privat: 0
      Msk: -1      # Never use for Msk (completely excluded)
      Chest: 1
    ct:
      Notfall: 0   # Only fallback for CT Notfall specifically

  YOU:
    default:
      Normal: 1
      Notfall: 1
      Herz: 0
      Privat: 0
      Msk: 1       # Msk specialist
      Chest: 0

  NUK:
    default:
      Normal: 0    # Only fallback
      Notfall: 0   # Only fallback
      Herz: 0
      Privat: -1   # Never for Privat
      Msk: -1
      Chest: -1
```

**Semantics:**
- `1` = Active (primary + fallback)
- `0` = Passive (fallback only)
- `-1` = Excluded (never use)

---

### **3. Time Calculation**

| Aspect | Approach A: Excel | Approach B: Config-Driven |
|--------|------------------|---------------------------|
| **Shift Times** | shifts.yml + complex logic | Can reuse shifts.yml OR config.yaml |
| **Friday Logic** | Hardcoded in compute_time_strings() | Can be in config or code |
| **Besonderheiten** | Hardcoded keyword matching | Label-based rules in config |
| **Format** | Semicolon-separated TIME → convert to VON/BIS | Direct VON/BIS |

**Both approaches need shift time calculation.** Approach B can reuse SBZDashboard's `shifts.yml` logic or simplify it.

---

### **4. Data Flow**

**Approach A:**
```python
# 1. Generate Excel
df = build_final_df(medweb_csv, date, config)
excel_path = generate_unified_excel(df)

# 2. Parse Excel (existing logic + multi-sheet support)
modality_data['ct'] = parse_excel(excel_path, sheet='ct')
modality_data['mr'] = parse_excel(excel_path, sheet='mr')
modality_data['xray'] = parse_excel(excel_path, sheet='xray')
```

**Approach B:**
```python
# Direct CSV → DataFrame
medweb_df = pd.read_csv(csv_path)
day_df = medweb_df[medweb_df['Datum'] == target_date]

# Apply mapping rules
for _, row in day_df.iterrows():
    rule = match_rule(row['Beschreibung der Aktivität'])
    modality = rule['modality']
    base_skills = rule['base_skills']

    # Apply roster overrides
    canonical_id = get_canonical_worker_id(row['PPL'])
    final_skills = apply_roster_overrides(base_skills, canonical_id, modality)

    # Compute time
    time_ranges = compute_time_strings(row, rule, config)

    # Add to modality DataFrame
    modality_data[modality].append({
        'PPL': row['PPL'],
        'start_time': ...,
        'end_time': ...,
        **final_skills
    })

# Convert to working_hours_df format
for mod in modality_data:
    modality_data[mod] = pd.DataFrame(modality_data[mod])
```

---

### **5. Modality Mapping**

**From medweb CSV analysis:**

| Medweb Activity | Old SBZDashboard | RadIMO Modality | Notes |
|----------------|------------------|-----------------|-------|
| CT Assistent, CT Spätdienst | CT | ct | ✅ Direct mapping |
| MR Assistent, MRT Spätdienst | MRT | mr | ✅ Direct mapping |
| SBZ: MRT-OA | MRT | mr | ✅ PP role |
| Chir Assistent, OA/FA Chir | Chir | xray | ✅ **Per user clarification** |
| SBZ OÄ/FA | ? | xray/mr/ct? | ❓ Needs decision |
| SBZ Aufklärung/Geräteassistenz | Special | ? | ❓ Label or separate modality? |

**User Clarification:** "chir=xray, he didnt now that" → Map all Chir activities to xray modality

---

### **6. Complexity**

| Metric | Approach A: Excel | Approach B: Config-Driven |
|--------|------------------|---------------------------|
| **Code Complexity** | High (preserve all SBZ logic) | Medium (config interpreter) |
| **Config Complexity** | Low | High (detailed mapping rules) |
| **Testing** | Excel generation + parsing | Direct data flow (easier) |
| **Debugging** | Excel as artifact (can inspect) | DataFrame in memory (need logging) |
| **Maintenance** | Code changes → redeploy | Config changes → restart |

---

### **7. Backward Compatibility**

| Aspect | Approach A: Excel | Approach B: Config-Driven |
|--------|------------------|---------------------------|
| **Manual Excel Upload** | ✅ Naturally supported (multi-sheet parser) | ✅ Keep existing upload route |
| **SBZ Excel Format** | ✅ Can generate legacy format | ⚠️ Excel export optional |
| **Migration Path** | Smooth (same format) | Config migration needed |

**Both approaches can support backward compatibility**, but Approach A has Excel as a natural format.

---

### **8. Extensibility**

**Scenario: Add new skill "Neuro"**

**Approach A:**
1. Update config.yaml skill_presets
2. Modify `build_final_df()` to handle Neuro logic
3. Update Excel generation templates
4. Redeploy

**Approach B:**
1. Update config.yaml skill_presets
2. Add Neuro to medweb_mapping base_skills
3. Add Neuro to worker_skill_roster for relevant workers
4. Restart (no code changes)

**Winner: Approach B** (config-driven, no code changes)

---

### **9. Implementation Effort**

**Approach A: Excel Generation**
| Task | Effort |
|------|--------|
| Port SBZDashboard code (build_final_df, shift logic) | 12-16h |
| Multi-sheet Excel generation | 4-6h |
| Multi-sheet Excel parsing | 6-8h |
| UI for schedule generator | 8-10h |
| Testing & integration | 6-8h |
| **Total** | **36-48h** |

**Approach B: Direct CSV Ingestion**
| Task | Effort |
|------|--------|
| Design medweb_mapping config structure | 4-6h |
| Design worker_skill_roster config structure | 4-6h |
| Implement build_working_hours_from_medweb() | 8-12h |
| Shift time calculation (reuse or simplify) | 6-8h |
| Config interpreter + roster override logic | 6-8h |
| UI for medweb upload | 4-6h |
| Testing & validation | 6-8h |
| **Total** | **38-54h** |

**Similar effort**, but Approach B has better long-term maintainability.

---

### **10. Pros & Cons Summary**

**Approach A: Excel Generation**

**Pros:**
- ✅ Preserves all SBZDashboard logic exactly
- ✅ Excel artifact for manual review/editing
- ✅ Easy backward compatibility
- ✅ Familiar format for admins
- ✅ Can export for use in other tools

**Cons:**
- ❌ More code to maintain (SBZDashboard logic + Excel generation)
- ❌ Excel as intermediate adds complexity
- ❌ Changes require code modifications
- ❌ Less transparent (logic in Python, not config)

---

**Approach B: Direct CSV Ingestion** ⭐

**Pros:**
- ✅ **Config-driven** (easier to extend/modify)
- ✅ **Cleaner architecture** (no Excel intermediate)
- ✅ **Transparent mapping** (all rules visible in config)
- ✅ **Roster system** (-1/0/1 overrides per worker/modality)
- ✅ **Less code** to maintain (rules in config, not code)
- ✅ **Easier debugging** (direct data flow)
- ✅ **Better for automation** (no file generation step)

**Cons:**
- ⚠️ No Excel artifact by default (can add optional export)
- ⚠️ Complex config structure (learning curve)
- ⚠️ Need to recreate SBZ logic (shift calculation, labels)

---

## 🎯 Recommendation

### **Use Approach B: Direct CSV Ingestion** ⭐

**Why:**

1. **Long-term Maintainability:**
   - Changes via config, not code
   - Transparent mapping rules
   - Easier to onboard new team members

2. **Better Architecture:**
   - Clean separation: mapping (config) vs logic (code)
   - No intermediate Excel step
   - Direct data flow (easier testing)

3. **Extensibility:**
   - Add new activities: update config
   - Add new workers: update roster
   - Add new skills: update presets + mappings

4. **User Clarification:**
   - "chir=xray" → Config handles modality aliasing naturally
   - Worker-specific overrides (Msk, Chest, Cardiac) fit roster model perfectly

5. **Optional Excel Export:**
   - Can still generate Excel for review/backup if needed
   - But not required for RadIMO to function

---

## 📐 Recommended Implementation Plan

### **Phase 1: Config-Driven Ingestion (MVP)** (2-3 weeks)

**Tasks:**

1. **Design config structure:**
   ```yaml
   medweb_mapping:
     rules:
       - match: "CT Spätdienst"
         modality: "ct"
         shift: "Spaetdienst"
         base_skills: {Normal: 1, Notfall: 1, ...}
       # ... more rules

   worker_skill_roster:
     KRUE:
       default: {Normal: 1, Notfall: 1, Msk: -1, ...}
       ct: {Notfall: 0}  # Per-modality override
     # ... more workers

   shift_times:
     Fruehdienst:
       default: "07:00-15:00"
       friday: "07:00-13:00"
     Spaetdienst:
       default: "13:00-21:00"
   ```

2. **Implement core function:**
   ```python
   def build_working_hours_from_medweb(
       csv_path: str,
       target_date: datetime.date,
       config: dict
   ) -> dict[str, pd.DataFrame]:
       """
       Parse medweb CSV and build working_hours_df for each modality.

       Returns:
           {
               'ct': DataFrame(PPL, start_time, end_time, Normal, Notfall, ...),
               'mr': DataFrame(...),
               'xray': DataFrame(...)  # includes Chir
           }
       """
       # Implementation here
   ```

3. **Update upload route:**
   ```python
   @app.route('/upload-medweb', methods=['POST'])
   @admin_required
   def upload_medweb():
       csv_file = request.files['csv']
       target_date = request.form['date']

       # Parse and load
       modality_dfs = build_working_hours_from_medweb(csv_file, target_date, APP_CONFIG)

       # Apply to modality_data
       for modality, df in modality_dfs.items():
           modality_data[modality]['working_hours_df'] = df
           # Initialize counters
           modality_data[modality]['draw_counts'] = {p: 0 for p in df['PPL']}
           # ...

       return jsonify({'success': True})
   ```

4. **Create UI:**
   - Add "Upload Medweb CSV" section to admin panel
   - Date picker for target date
   - Preview parsed data before applying
   - Keep existing Excel upload for backward compatibility

---

### **Phase 2: Advanced Features** (1-2 weeks)

1. **Label-based logic:**
   - Besonderheiten keywords → skill modifications
   - "Cardiac" label → Herz=1
   - "Aufklärung" label → time modification

2. **Worker roster management:**
   - UI for editing worker_skill_roster
   - Validate roster entries
   - Export/import roster as CSV

3. **Optional Excel export:**
   ```python
   @app.route('/export-excel/<modality>')
   def export_excel(modality):
       df = modality_data[modality]['working_hours_df']
       excel_path = generate_backup_excel(df, modality)
       return send_file(excel_path)
   ```

4. **Config validation:**
   - Check medweb_mapping rules (all skills present, valid modalities)
   - Check worker_skill_roster (valid workers, valid skills)
   - Warn about missing mappings

---

### **Phase 3: Migration & Testing** (1 week)

1. **Create migration script:**
   ```python
   # Convert old config.yml + shifts.yml → new config.yaml
   def migrate_sbz_config(old_config_path, old_shifts_path):
       # Extract skill_overrides → worker_skill_roster
       # Extract shifts.yml → shift_times
       # Generate medweb_mapping rules from known patterns
   ```

2. **Compare outputs:**
   - Run SBZDashboard on medweb CSV → Excel
   - Run new ingestion on same CSV → working_hours_df
   - Compare results (PPL, times, skills)
   - Fix discrepancies

3. **Testing:**
   - Unit tests for build_working_hours_from_medweb()
   - Integration tests with real medweb CSV
   - Edge cases (1. Monat, PP roles, Besonderheiten, overlapping shifts)

---

## 🔧 Technical Details for Approach B

### **1. Mapping Rule Matching**

```python
def match_mapping_rule(activity_desc: str, rules: list) -> dict | None:
    """Find first matching rule for activity description."""
    for rule in rules:
        match_str = rule.get('match', '')
        if match_str.lower() in activity_desc.lower():
            return rule
    return None
```

### **2. Roster Override Application**

```python
def apply_roster_overrides(
    base_skills: dict,
    canonical_id: str,
    modality: str,
    worker_roster: dict
) -> dict:
    """Apply per-worker skill overrides."""
    if canonical_id not in worker_roster:
        return base_skills

    # Start with base skills
    final_skills = base_skills.copy()

    # Apply default overrides
    if 'default' in worker_roster[canonical_id]:
        final_skills.update(worker_roster[canonical_id]['default'])

    # Apply modality-specific overrides
    if modality in worker_roster[canonical_id]:
        final_skills.update(worker_roster[canonical_id][modality])

    return final_skills
```

### **3. Shift Time Calculation**

```python
def compute_time_ranges(
    row: pd.Series,
    rule: dict,
    target_date: datetime.date,
    config: dict
) -> list[tuple[datetime.time, datetime.time]]:
    """
    Compute start/end times based on shift and labels.

    Can reuse SBZDashboard's shifts.yml logic or simplify.
    """
    shift_name = rule.get('shift', 'Fruehdienst')
    shift_config = config['shift_times'][shift_name]

    # Check for special days (Friday, etc.)
    is_friday = target_date.weekday() == 4

    if is_friday and 'friday' in shift_config:
        time_str = shift_config['friday']
    else:
        time_str = shift_config['default']

    # Parse "07:00-15:00" → (time(7,0), time(15,0))
    start_str, end_str = time_str.split('-')
    start_time = datetime.strptime(start_str, '%H:%M').time()
    end_time = datetime.strptime(end_str, '%H:%M').time()

    # Handle labels (Aufklärung, etc.)
    labels = row.get('Beschreibung der Aktivität', '')
    if 'Aufklärung' in labels:
        # Modify times based on label
        pass

    return [(start_time, end_time)]
```

### **4. Complete Pipeline**

```python
def build_working_hours_from_medweb(
    csv_path: str,
    target_date: datetime.date,
    config: dict
) -> dict[str, pd.DataFrame]:

    # 1. Load CSV
    medweb_df = pd.read_csv(csv_path, sep=',', encoding='latin1')
    medweb_df['Datum_parsed'] = pd.to_datetime(
        medweb_df['Datum'], dayfirst=True
    ).dt.date

    day_df = medweb_df[medweb_df['Datum_parsed'] == target_date]

    if day_df.empty:
        return {}

    # 2. Prepare data structures
    mapping_rules = config.get('medweb_mapping', {}).get('rules', [])
    worker_roster = config.get('worker_skill_roster', {})

    rows_per_modality = {mod: [] for mod in allowed_modalities}

    # 3. Process each activity
    for _, row in day_df.iterrows():
        activity_desc = str(row['Beschreibung der Aktivität'])

        # Match rule
        rule = match_mapping_rule(activity_desc, mapping_rules)
        if not rule:
            continue  # Not SBZ-relevant

        modality = normalize_modality(rule['modality'])
        base_skills = {s: 0 for s in SKILL_COLUMNS}
        base_skills.update(rule.get('base_skills', {}))

        # Apply Msk/Chest from ID lists if needed
        ppl_str = build_ppl(row)
        canonical_id = get_canonical_worker_id(ppl_str)

        # Apply roster overrides
        final_skills = apply_roster_overrides(
            base_skills, canonical_id, modality, worker_roster
        )

        # Compute time ranges
        time_ranges = compute_time_ranges(row, rule, target_date, config)

        # Add row(s) for each time range
        for start_time, end_time in time_ranges:
            rows_per_modality[modality].append({
                'PPL': ppl_str,
                'canonical_id': canonical_id,
                'start_time': start_time,
                'end_time': end_time,
                'shift_duration': (
                    datetime.combine(target_date, end_time) -
                    datetime.combine(target_date, start_time)
                ).seconds / 3600,
                'Modifier': 1.0,  # Can add modifier_overrides from config
                **final_skills
            })

    # 4. Convert to DataFrames
    result = {}
    for modality, rows in rows_per_modality.items():
        if not rows:
            continue
        df = pd.DataFrame(rows)
        result[modality] = df

    return result
```

---

## 📝 Example Config for Approach B

**config.yaml (extended):**

```yaml
# Existing modalities
modalities:
  ct:
    label: CT
    factor: 1.0
    color: '#3498db'
  mr:
    label: MR
    factor: 1.2
    color: '#9b59b6'
  xray:
    label: Röntgen/Chir  # Note: includes Chir
    factor: 1.0
    color: '#e74c3c'

# Existing skills
skills:
  - {name: Normal, label: Normal, color: '#2ecc71'}
  - {name: Notfall, label: Notfall, color: '#e74c3c'}
  - {name: Privat, label: Privatpatienten, color: '#f39c12'}
  - {name: Herz, label: Herz/Kardio, color: '#e91e63'}
  - {name: Msk, label: MSK, color: '#00bcd4'}
  - {name: Chest, label: Chest, color: '#795548'}

# NEW: Medweb activity mapping
medweb_mapping:
  rules:
    # CT activities
    - match: "CT Spätdienst"
      modality: "ct"
      shift: "Spaetdienst"
      base_skills: {Normal: 1, Notfall: 1, Herz: 0, Privat: 0, Msk: 0, Chest: 0}

    - match: "CT Assistent"
      modality: "ct"
      shift: "Fruehdienst"
      base_skills: {Normal: 1, Notfall: 1, Herz: 0, Privat: 0, Msk: 0, Chest: 0}

    - match: "SBZ: CT Privatpatienten"
      modality: "ct"
      shift: "Fruehdienst"
      base_skills: {Normal: 0, Notfall: 0, Herz: 0, Privat: 1, Msk: 0, Chest: 0}

    # MR activities
    - match: "MRT Spätdienst"
      modality: "mr"
      shift: "Spaetdienst"
      base_skills: {Normal: 1, Notfall: 1, Herz: 0, Privat: 0, Msk: 0, Chest: 0}

    - match: "MR Assistent 1. Monat"
      modality: "mr"
      shift: "Fruehdienst"
      base_skills: {Normal: 1, Notfall: 0, Herz: 0, Privat: 0, Msk: 0, Chest: 0}

    - match: "MR Assistent"
      modality: "mr"
      shift: "Fruehdienst"
      base_skills: {Normal: 1, Notfall: 1, Herz: 0, Privat: 0, Msk: 0, Chest: 0}

    - match: "SBZ: MRT-OA"
      modality: "mr"
      shift: "Fruehdienst"
      base_skills: {Normal: 0, Notfall: 0, Herz: 0, Privat: 1, Msk: 0, Chest: 0}

    # Chir activities → xray modality per user clarification
    - match: "Chir Assistent"
      modality: "xray"
      shift: "Fruehdienst"
      base_skills: {Normal: 1, Notfall: 1, Herz: 0, Privat: 0, Msk: 0, Chest: 0}

    - match: "OA / FA Chir"
      modality: "xray"
      shift: "Fruehdienst"
      base_skills: {Normal: 1, Notfall: 1, Herz: 0, Privat: 1, Msk: 0, Chest: 0}

    # Special activities
    - match: "SBZ OÄ/FA"  # CSV encoding: "SBZ OÃ/FÃ"
      modality: "xray"  # Or could be multi-modality? Needs clarification
      shift: "Fruehdienst"
      base_skills: {Normal: 0, Notfall: 0, Herz: 0, Privat: 1, Msk: 0, Chest: 0}

    - match: "SBZ Aufklärung/Geräteassistenz"
      modality: "ct"  # Or special handling?
      shift: "Fruehdienst"
      label: "Aufklärung"  # Can use for time modification
      base_skills: {Normal: 1, Notfall: 0, Herz: 0, Privat: 0, Msk: 0, Chest: 0}

# NEW: Shift time definitions
shift_times:
  Fruehdienst:
    default: "07:00-15:00"
    friday: "07:00-13:00"
  Spaetdienst:
    default: "13:00-21:00"
    friday: "13:00-19:00"

# NEW: Worker skill roster (per-worker overrides)
worker_skill_roster:
  # Example workers with specific skill profiles
  KRUE:
    default:
      Normal: 1
      Notfall: 1
      Herz: 0
      Privat: 0
      Msk: -1     # Never Msk
      Chest: 1    # Chest specialist
    ct:
      Notfall: 0  # Only fallback for CT Notfall

  YOU:
    default:
      Normal: 1
      Notfall: 1
      Herz: 0
      Privat: 0
      Msk: 1      # Msk specialist
      Chest: 0

  NUK:
    default:
      Normal: 0   # Only fallback
      Notfall: 0  # Only fallback
      Herz: 0
      Privat: -1  # Never Privat
      Msk: -1
      Chest: -1

  # Add more workers as needed
  # Can also reference by Personalnummer if needed

# Existing settings
fallback_mode: skill_priority
fallback_threshold_percent: 30
skill_weight_modifier: 1.25
```

---

## 🚀 Next Steps

1. **Get user confirmation:**
   - Approve Approach B (config-driven)?
   - Confirm chir → xray mapping
   - Clarify "SBZ OÄ/FA" modality
   - Clarify "Aufklärung" handling

2. **Create config prototype:**
   - Draft complete medweb_mapping rules from sample CSV
   - Draft worker_skill_roster for known workers
   - Validate shift_times structure

3. **Implement Phase 1:**
   - Build `build_working_hours_from_medweb()` function
   - Add medweb upload route
   - Create preview UI
   - Test with real medweb CSV

4. **Testing:**
   - Compare outputs with SBZDashboard
   - Validate all activity types
   - Test roster overrides
   - Test edge cases

---

**Document Version:** 2.0
**Date:** 2025-11-20
**Status:** Awaiting User Confirmation

