# Excel Upload Path Migration Strategy

**Decision Document: What to do with the old Excel-driven upload workflow**

---

## 📋 Context

**Current State:**
- RadIMO accepts manual Excel file uploads (one file per modality: SBZ_ct.xlsx, SBZ_mr.xlsx, SBZ_xray.xlsx)
- Each file has Tabelle1 (worker data) and Tabelle2 (info texts)
- Format: PPL, Modifier, VON, BIS, Normal, Notfall, Herz, Privat, Msk, Chest (-1/0/1 values)

**New Approach:**
- Direct medweb CSV ingestion with config-driven mapping
- Single upload → all modalities populated
- No manual Excel creation needed

**Question:** What should we do with the old Excel upload path?

---

## 🎯 Four Options

### **Option A: Remove Excel Upload Completely** 🔥

**Changes:**
- Delete `/upload` route and upload.html template
- Remove all Excel parsing code (parse_roster, parse_time_range, etc.)
- Keep only medweb CSV upload
- Simplify codebase significantly

**Pros:**
- ✅ **Cleanest codebase** (no legacy code)
- ✅ **Single source of truth** (medweb CSV only)
- ✅ **Reduced maintenance** (one upload path, not two)
- ✅ **Forces migration** (users must adopt new system)
- ✅ **No confusion** (only one way to do things)
- ✅ **Smaller code footprint** (~500-800 lines removed)

**Cons:**
- ❌ **No fallback** if medweb CSV unavailable
- ❌ **Breaking change** for existing users
- ❌ **No manual override** capability
- ❌ **Risky deployment** (all-or-nothing)
- ❌ **Loss of flexibility** (can't handle edge cases manually)

**User Impact:**
- **HIGH** - Must immediately switch to medweb CSV
- All existing Excel files become unusable
- No gradual transition period

**Risk Level:** 🔴 **HIGH**

---

### **Option B: Keep Excel, Hide by Default** 🔒

**Changes:**
- Keep Excel upload code but hide UI behind feature flag
- Add config setting: `enable_legacy_excel_upload: false`
- Admin can enable if needed for emergency/manual override
- New users see only medweb CSV upload

**Implementation:**
```yaml
# config.yaml
features:
  legacy_excel_upload: false  # Hidden by default
  medweb_csv_upload: true     # New default
```

```python
# app.py
@app.route('/upload')
@admin_required
def upload_file():
    legacy_enabled = APP_CONFIG.get('features', {}).get('legacy_excel_upload', False)
    return render_template('upload.html',
                         legacy_excel_enabled=legacy_enabled,
                         medweb_csv_enabled=True)
```

**Pros:**
- ✅ **Safety net** (Excel upload still works if enabled)
- ✅ **Manual override capability** (for edge cases)
- ✅ **Graceful migration** (can re-enable if issues)
- ✅ **Emergency fallback** (if medweb CSV system fails)
- ✅ **Clean default UX** (new users don't see legacy)

**Cons:**
- ⚠️ **Maintenance burden** (must maintain both code paths)
- ⚠️ **Complexity** (two upload mechanisms)
- ⚠️ **Testing overhead** (test both paths)
- ⚠️ **Code bloat** (legacy code stays)

**User Impact:**
- **LOW** - Default is new system, legacy hidden
- Power users can enable if needed
- Smooth transition

**Risk Level:** 🟡 **MEDIUM**

---

### **Option C: Keep Both Paths Active** 🔄

**Changes:**
- Keep both Excel and medweb CSV upload
- Show both in UI with clear labels
- Let users choose which to use
- No deprecation timeline

**Implementation:**
```html
<!-- upload.html -->
<div class="upload-section">
  <h2>Option 1: Upload Medweb CSV (Recommended)</h2>
  <form method="POST" action="/upload-medweb">
    <input type="file" name="csv" accept=".csv">
    <input type="date" name="date">
    <button type="submit">Upload CSV</button>
  </form>
</div>

<div class="upload-section">
  <h2>Option 2: Manual Excel Upload (Per Modality)</h2>
  <form method="POST" action="/upload-excel">
    <select name="modality">
      <option>ct</option>
      <option>mr</option>
      <option>xray</option>
    </select>
    <input type="file" name="excel" accept=".xlsx">
    <button type="submit">Upload Excel</button>
  </form>
</div>
```

**Pros:**
- ✅ **Maximum flexibility** (users choose)
- ✅ **Zero breaking changes** (everything still works)
- ✅ **Handles all edge cases** (medweb + manual overrides)
- ✅ **Easy deployment** (no migration needed)
- ✅ **Supports mixed workflows** (some modalities via CSV, some via Excel)

**Cons:**
- ❌ **Confusing UX** (two ways to do same thing)
- ❌ **Maintenance burden** (both paths forever)
- ❌ **No migration pressure** (users may never switch)
- ❌ **Testing complexity** (interactions between paths)
- ❌ **Code bloat** (both systems maintained)

**User Impact:**
- **NONE** - Everything works as before + new option
- May cause confusion about which to use

**Risk Level:** 🟢 **LOW** (but technical debt HIGH)

---

### **Option D: Deprecation with Migration Period** ⏱️

**Changes:**
- Keep both paths for defined period (e.g., 3-6 months)
- Show deprecation warning on Excel upload
- Track usage to understand adoption
- Remove Excel upload after migration period

**Implementation:**
```python
# config.yaml
features:
  legacy_excel_upload: true
  legacy_excel_deprecation_date: "2025-06-01"  # Remove after this date
  medweb_csv_upload: true
```

```html
<!-- upload.html - Excel section -->
<div class="upload-section deprecated">
  <div class="warning-banner">
    ⚠️ Excel upload will be removed on June 1, 2025.
    Please migrate to medweb CSV upload.
  </div>
  <form method="POST" action="/upload-excel">
    <!-- existing form -->
  </form>
</div>
```

```python
# Track usage
@app.route('/upload-excel', methods=['POST'])
def upload_excel():
    log_usage('legacy_excel_upload', user=current_user)

    # Check if past deprecation date
    deprecation_date = APP_CONFIG['features']['legacy_excel_deprecation_date']
    if datetime.now().date() > datetime.strptime(deprecation_date, '%Y-%m-%d').date():
        return jsonify({'error': 'Excel upload has been removed. Please use medweb CSV upload.'}), 410

    # Proceed with upload...
```

**Pros:**
- ✅ **Clear migration path** (users know timeline)
- ✅ **Safety net during transition** (both work)
- ✅ **Usage tracking** (understand adoption)
- ✅ **Eventual cleanup** (legacy code removed)
- ✅ **User-friendly** (time to adapt)
- ✅ **Reduced long-term debt** (temporary dual maintenance)

**Cons:**
- ⚠️ **Temporary complexity** (maintain both for period)
- ⚠️ **Requires communication** (inform users of deadline)
- ⚠️ **Potential pushback** (forced migration)

**User Impact:**
- **MEDIUM** - Must migrate within timeframe
- Clear warning and deadline
- Time to test new system

**Risk Level:** 🟡 **MEDIUM** (well-managed)

---

## 📊 Detailed Comparison Matrix

| Criterion | A: Remove | B: Hide | C: Keep Both | D: Deprecate |
|-----------|-----------|---------|--------------|--------------|
| **Code Complexity** | ⭐⭐⭐⭐⭐ Minimal | ⭐⭐⭐⭐ Low | ⭐⭐ High | ⭐⭐⭐ Medium |
| **Maintenance Burden** | ⭐⭐⭐⭐⭐ Minimal | ⭐⭐⭐ Medium | ⭐ High | ⭐⭐⭐⭐ Low (after period) |
| **User Impact** | ⭐ Very High | ⭐⭐⭐⭐ Low | ⭐⭐⭐⭐⭐ None | ⭐⭐⭐ Medium |
| **Risk** | ⭐ High | ⭐⭐⭐ Medium | ⭐⭐⭐⭐⭐ Low | ⭐⭐⭐⭐ Low |
| **Flexibility** | ⭐ None | ⭐⭐⭐⭐ High | ⭐⭐⭐⭐⭐ Maximum | ⭐⭐⭐⭐ High (temp) |
| **Migration Pressure** | ⭐⭐⭐⭐⭐ Forces | ⭐⭐⭐ Encourages | ⭐ None | ⭐⭐⭐⭐ Clear timeline |
| **Emergency Fallback** | ❌ None | ✅ Available | ✅ Available | ✅ Available (temp) |
| **Long-term Health** | ⭐⭐⭐⭐⭐ Best | ⭐⭐⭐ Good | ⭐ Poor | ⭐⭐⭐⭐⭐ Best |
| **Testing Burden** | ⭐⭐⭐⭐⭐ Minimal | ⭐⭐⭐ Medium | ⭐ High | ⭐⭐⭐ Medium (temp) |
| **Documentation Effort** | ⭐⭐⭐ Medium | ⭐⭐⭐⭐ Low | ⭐⭐ High | ⭐⭐⭐⭐ Low |

---

## 🔍 Deep Dive: Technical Impact

### **Code Sections Affected**

**If Excel upload removed (Option A):**

```python
# DELETE these functions (~600 lines):
def parse_roster()           # Excel parsing
def parse_time_range()       # TIME column parsing
def backup_dataframe()       # Excel export
def allowed_file()           # File validation
def save_uploaded_file()     # File handling
@app.route('/upload')        # Upload route
@app.route('/upload-file')   # Per-modality upload

# DELETE templates:
templates/upload.html        # Upload interface (~800 lines)

# KEEP these (needed for medweb):
def get_canonical_worker_id()
def normalize_modality()
modality_data structure
SKILL_COLUMNS
```

**If Excel kept (Options B/C/D):**

```python
# KEEP all existing code
# ADD new code:
def build_working_hours_from_medweb()  # ~150 lines
def match_mapping_rule()                # ~20 lines
def apply_roster_overrides()            # ~30 lines
def compute_time_ranges()               # ~50 lines
@app.route('/upload-medweb')            # ~50 lines

# MODIFY:
templates/upload.html  # Add medweb section (~200 lines)
config.yaml            # Add mapping rules (~100 lines)
```

**Net Code Impact:**

| Option | Lines Added | Lines Removed | Net Change | Total Codebase |
|--------|-------------|---------------|------------|----------------|
| **A: Remove** | ~250 (medweb) | ~1400 (Excel) | **-1150** | Smaller ✅ |
| **B: Hide** | ~250 (medweb) | 0 | **+250** | Larger ⚠️ |
| **C: Keep Both** | ~250 (medweb) | 0 | **+250** | Larger ⚠️ |
| **D: Deprecate** | ~250 (medweb) | ~1400 (later) | **+250 → -1150** | Smaller ✅ (eventually) |

---

## 🎯 Use Cases & Scenarios

### **Scenario 1: Normal Daily Operation**

**With Excel (Current):**
1. Medweb exports CSV monthly
2. User runs SBZDashboard → generates 3 Excel files
3. User manually uploads each Excel to RadIMO (3 uploads)
4. RadIMO parses and loads data

**With Medweb CSV (New):**
1. Medweb exports CSV monthly
2. User uploads CSV once to RadIMO
3. RadIMO parses via config mapping → loads all modalities

**Winner:** Medweb CSV (fewer steps)

---

### **Scenario 2: Manual Override Needed**

**Example:** Worker calls in sick after schedule generated, need to adjust skills/times

**Option A (Remove Excel):**
- ❌ **No manual override path**
- Must modify medweb CSV and re-upload
- Or manually edit working_hours_df in database (risky)

**Option B/C/D (Keep Excel):**
- ✅ **Excel override available**
- Generate Excel from current state
- Manually edit Excel
- Re-upload just affected modality

**Winner:** Options B/C/D (flexibility)

---

### **Scenario 3: Medweb System Down**

**Example:** Medweb CSV export broken, need to create schedule manually

**Option A (Remove Excel):**
- ❌ **No fallback**
- Must wait for medweb fix
- Or manually create CSV matching medweb format

**Option B/C/D (Keep Excel):**
- ✅ **Fallback available**
- Create Excel manually from scratch
- RadIMO continues working

**Winner:** Options B/C/D (resilience)

---

### **Scenario 4: Other Department Wants to Use RadIMO**

**Example:** Neuro department wants to use RadIMO but doesn't have medweb integration yet

**Option A (Remove Excel):**
- ❌ **Cannot use RadIMO**
- Must first integrate with medweb
- High barrier to entry

**Option B/C/D (Keep Excel):**
- ✅ **Can use Excel upload**
- Works immediately
- Can migrate to medweb later

**Winner:** Options B/C/D (extensibility)

---

### **Scenario 5: Testing New Config Changes**

**Example:** Admin wants to test new medweb_mapping rules before going live

**Option A (Remove Excel):**
- Test on production with medweb CSV
- Risk breaking live system
- Or need separate test environment

**Option B/C/D (Keep Excel):**
- Test medweb CSV on subset
- Keep Excel as backup
- Can rollback easily

**Winner:** Options B/C/D (safety)

---

## 🏗️ Architecture Considerations

### **Current RadIMO Data Flow**

```
Excel Upload
    ↓
parse_roster()
    ↓
modality_data['ct']['working_hours_df'] = DataFrame(PPL, start_time, end_time, skills...)
modality_data['ct']['draw_counts'] = {worker: 0}
modality_data['ct']['skill_counts'] = {skill: {worker: 0}}
    ↓
Worker Assignment Logic
    ↓
_should_balance_via_fallback()
_attempt_column_selection()
select_worker_from_pool()
```

### **New Medweb CSV Flow**

```
Medweb CSV Upload
    ↓
build_working_hours_from_medweb()
├── match_mapping_rule() (config.yaml rules)
├── apply_roster_overrides() (config.yaml roster)
└── compute_time_ranges() (shifts.yml logic)
    ↓
modality_data['ct']['working_hours_df'] = DataFrame(PPL, start_time, end_time, skills...)
    ↓
[SAME] Worker Assignment Logic (unchanged)
```

### **Key Insight:**

**Both paths produce the same data structure** → `working_hours_df`

This means:
- ✅ Assignment logic doesn't care about source
- ✅ Can mix sources (CT from CSV, MR from Excel)
- ✅ Easy to keep both paths (they're independent until final DataFrame)

**Implication:** Option C (Keep Both) has low technical risk, but high maintenance burden.

---

## 🛡️ Risk Analysis

### **Option A (Remove Excel): Risks**

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Medweb CSV unavailable | Medium | High | None - system breaks |
| Manual override needed | High | Medium | None - can't do it |
| Other dept wants to use | Medium | High | Must integrate medweb first |
| Config mapping bug | Medium | High | No fallback |
| User resistance | High | Medium | Communication, training |

**Overall Risk: HIGH** 🔴

---

### **Option B (Hide Excel): Risks**

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Medweb CSV unavailable | Medium | Low | Enable Excel flag |
| Manual override needed | High | Low | Enable Excel flag |
| Config mapping bug | Medium | Low | Fallback to Excel |
| Users find hidden flag | Low | Low | Document it clearly |
| Maintenance burden | High | Medium | Code is stable |

**Overall Risk: LOW** 🟢

---

### **Option C (Keep Both): Risks**

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| User confusion | High | Medium | Clear UI labels |
| Both paths out of sync | Medium | Medium | Choose one as source of truth |
| Testing gaps | High | Medium | Test both paths |
| Code drift | High | High | Code reviews, CI |
| Users never migrate | High | High | None - no pressure |

**Overall Risk: MEDIUM** 🟡 (technical debt HIGH)

---

### **Option D (Deprecate): Risks**

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Users not ready by deadline | Medium | Medium | Extend deadline if needed |
| Medweb issues discovered late | Low | Medium | Monitor during period |
| Pushback from users | Medium | Low | Clear communication |
| Premature removal | Low | High | Track usage before removal |

**Overall Risk: LOW** 🟢

---

## 💼 Business Considerations

### **Team Resources**

**Option A (Remove):**
- Initial effort: 2-3 days (implement medweb, remove Excel code, testing)
- Ongoing: Minimal (single code path)
- **Best if:** Small team, limited resources

**Option B (Hide):**
- Initial effort: 3-4 days (implement medweb, add feature flag, testing)
- Ongoing: Low (Excel code stable, rarely touched)
- **Best if:** Want safety net, medium team

**Option C (Keep Both):**
- Initial effort: 3-4 days (implement medweb, integrate both paths)
- Ongoing: High (maintain both, test both)
- **Best if:** Large team, diverse user base

**Option D (Deprecate):**
- Initial effort: 4-5 days (implement medweb, add warnings, tracking)
- Phase 2 effort: 1-2 days (remove Excel after period)
- Ongoing: Medium → Low
- **Best if:** Want clean eventual state, willing to manage transition

---

### **User Base Considerations**

**If single site (Freiburg only):**
- ✅ Option A or D (can control migration)
- ⚠️ Option B (if want emergency fallback)
- ❌ Option C (unnecessary flexibility)

**If multi-site deployment:**
- ❌ Option A (too risky)
- ✅ Option B or C (sites at different stages)
- ✅ Option D (with per-site tracking)

**If other departments may adopt:**
- ❌ Option A (blocks adoption)
- ✅ Option B (hidden but available)
- ✅ Option C (each dept chooses)
- ✅ Option D (with extended period)

---

## 📈 Migration Strategies

### **For Option A (Remove Excel)**

**Pre-Migration:**
1. ✅ Implement medweb CSV upload fully
2. ✅ Test extensively with real data
3. ✅ Create comprehensive config.yaml with all mappings
4. ✅ Document new workflow
5. ✅ Train users on medweb upload

**Migration Day:**
1. Deploy new version with Excel code removed
2. Monitor for issues
3. Be ready to rollback if critical issues

**Post-Migration:**
1. Remove Excel-related documentation
2. Simplify codebase
3. Archive Excel templates

**Timeline:** 1 week prep + 1 day migration

---

### **For Option B (Hide Excel)**

**Implementation:**
1. Add feature flag to config.yaml
2. Conditional UI rendering in upload.html
3. Add admin note about hidden feature
4. Update documentation

**No migration needed** - both paths work

**Timeline:** 1-2 days

---

### **For Option C (Keep Both)**

**Implementation:**
1. Add medweb upload section to upload.html
2. Keep Excel upload section
3. Add clear labels/recommendations
4. Document both paths

**No migration needed** - both paths work

**Timeline:** 1-2 days

---

### **For Option D (Deprecate)**

**Phase 1: Deprecation Notice (Month 1-2)**
1. Deploy medweb CSV upload
2. Add warning banner to Excel upload
3. Communicate deprecation timeline to users
4. Track usage of both paths

**Phase 2: Migration Period (Month 3-5)**
1. Monitor medweb adoption rate
2. Provide support for migration issues
3. Send reminders as deadline approaches
4. Extend deadline if needed (based on usage data)

**Phase 3: Removal (Month 6)**
1. Final warning (1 week before)
2. Remove Excel upload code
3. Update UI to only show medweb
4. Monitor for issues

**Timeline:** 6 months total

---

## 🎯 Recommendation

### **Primary Recommendation: Option D (Deprecation with Migration Period)** ⭐

**Why:**

1. **Balanced Risk/Reward:**
   - ✅ Safety net during transition
   - ✅ Clear path to clean codebase
   - ✅ Time for users to adapt
   - ✅ Usage tracking informs decision

2. **Best Long-Term Outcome:**
   - Eventually clean codebase (like Option A)
   - But with managed risk (like Option B)

3. **User-Friendly:**
   - No sudden breaking changes
   - Clear expectations
   - Time to test new system

4. **Flexible:**
   - Can extend deadline if needed
   - Can gather feedback during period
   - Can keep Excel if critical issues found

**Timeline:**
- **Month 1-2:** Deploy medweb, add deprecation warnings, communicate
- **Month 3-5:** Migration period, support, monitoring
- **Month 6:** Remove Excel upload code

---

### **Alternative Recommendation: Option B (Hide Excel)** ⭐

**When to choose:**

1. **If multi-site deployment** - Different sites may need different approaches
2. **If emergency fallback critical** - Medical/critical systems
3. **If other departments want to adopt** - Each dept may have different data sources
4. **If team resources limited** - Can't manage 6-month deprecation process

**Why it's good:**
- ✅ Minimal disruption
- ✅ Maximum flexibility
- ✅ Low risk
- ⚠️ Small maintenance burden (Excel code is stable)

---

### **NOT Recommended:**

**Option A (Remove):** Too risky without transition period

**Option C (Keep Both):** Creates long-term technical debt with no migration pressure

---

## 📝 Implementation Plan for Option D (Recommended)

### **Phase 1: Deploy Medweb CSV (Week 1-2)**

**Code Changes:**
```python
# config.yaml
features:
  medweb_csv_upload: true
  legacy_excel_upload: true  # Will be removed 2025-06-01
  legacy_excel_deprecation_date: "2025-06-01"

medweb_mapping:
  rules:
    # ... (from INTEGRATION_COMPARISON.md)

worker_skill_roster:
  # ... (from INTEGRATION_COMPARISON.md)

shift_times:
  # ... (from shifts.yml or simplified)
```

```python
# app.py - Add new route
@app.route('/upload-medweb', methods=['POST'])
@admin_required
def upload_medweb():
    csv_file = request.files['csv']
    target_date = request.form.get('date')

    # Log usage
    log_usage('medweb_csv_upload', user=session.get('admin'))

    # Parse and load
    try:
        modality_dfs = build_working_hours_from_medweb(
            csv_file,
            datetime.strptime(target_date, '%Y-%m-%d').date(),
            APP_CONFIG
        )

        # Apply to modality_data
        for modality, df in modality_dfs.items():
            modality_data[modality]['working_hours_df'] = df
            modality_data[modality]['draw_counts'] = {p: 0 for p in df['PPL'].unique()}
            modality_data[modality]['skill_counts'] = {
                skill: {p: 0 for p in df['PPL'].unique()}
                for skill in SKILL_COLUMNS
            }
            modality_data[modality]['WeightedCounts'] = {p: 0.0 for p in df['PPL'].unique()}

        return jsonify({
            'success': True,
            'modalities_loaded': list(modality_dfs.keys()),
            'total_workers': sum(len(df) for df in modality_dfs.values())
        })

    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 400

# Modify existing Excel upload route
@app.route('/upload-excel', methods=['POST'])
@admin_required
def upload_excel():
    # Log usage
    log_usage('legacy_excel_upload', user=session.get('admin'))

    # Check deprecation date
    deprecation_date_str = APP_CONFIG['features'].get('legacy_excel_deprecation_date')
    if deprecation_date_str:
        deprecation_date = datetime.strptime(deprecation_date_str, '%Y-%m-%d').date()
        if datetime.now().date() >= deprecation_date:
            return jsonify({
                'success': False,
                'error': 'Excel upload has been removed. Please use medweb CSV upload.'
            }), 410  # HTTP 410 Gone

    # Existing Excel upload logic...
    # [keep existing code]
```

```html
<!-- templates/upload.html - Add sections -->
<div class="upload-section recommended">
  <h2>✅ Medweb CSV Upload (Recommended)</h2>
  <p>Upload a single medweb CSV file to populate all modalities.</p>

  <form id="medweb-upload-form" method="POST" action="/upload-medweb" enctype="multipart/form-data">
    <div class="form-group">
      <label>Medweb CSV File:</label>
      <input type="file" name="csv" accept=".csv" required>
    </div>

    <div class="form-group">
      <label>Target Date:</label>
      <input type="date" name="date" required>
    </div>

    <button type="submit" class="btn-primary">Upload & Parse CSV</button>
  </form>

  <div id="medweb-preview" style="display:none;">
    <!-- Preview parsed data here -->
  </div>
</div>

<hr>

<div class="upload-section deprecated">
  <div class="deprecation-warning">
    ⚠️ <strong>Deprecation Notice:</strong> Excel upload will be removed on <strong>June 1, 2025</strong>.
    Please migrate to medweb CSV upload above.
    <br>
    <small>Reason: Medweb CSV is faster, config-driven, and supports all modalities in one upload.</small>
  </div>

  <h2>📋 Excel Upload (Per Modality) - DEPRECATED</h2>
  <p>This upload method will be removed. Please use medweb CSV upload instead.</p>

  <!-- Existing Excel upload form -->
  <!-- [keep existing code] -->
</div>

<style>
.deprecated {
  opacity: 0.7;
  border: 2px dashed #f39c12;
  padding: 1rem;
  border-radius: 4px;
}

.deprecation-warning {
  background: #fff3cd;
  border: 1px solid #f39c12;
  padding: 1rem;
  border-radius: 4px;
  margin-bottom: 1rem;
  color: #856404;
}

.recommended {
  border: 2px solid #27ae60;
  padding: 1rem;
  border-radius: 4px;
  background: #d4edda;
}
</style>
```

**Testing:**
1. Test medweb CSV upload with sample data
2. Verify all modalities load correctly
3. Test worker assignment still works
4. Verify Excel upload still works (with warning)

---

### **Phase 2: Monitor & Support (Month 1-5)**

**Tracking:**
```python
# Add usage tracking
usage_log = []  # Or database table

def log_usage(method: str, user: str):
    usage_log.append({
        'timestamp': datetime.now(),
        'method': method,  # 'medweb_csv_upload' or 'legacy_excel_upload'
        'user': user
    })

# Add admin dashboard endpoint
@app.route('/admin/usage-stats')
@admin_required
def usage_stats():
    # Aggregate usage data
    total_medweb = sum(1 for log in usage_log if log['method'] == 'medweb_csv_upload')
    total_excel = sum(1 for log in usage_log if log['method'] == 'legacy_excel_upload')

    return jsonify({
        'medweb_uploads': total_medweb,
        'excel_uploads': total_excel,
        'adoption_rate': total_medweb / (total_medweb + total_excel) if (total_medweb + total_excel) > 0 else 0
    })
```

**Monthly Review:**
- Check adoption rate
- Gather user feedback
- Address migration blockers
- Extend deadline if needed

---

### **Phase 3: Remove Excel Upload (Month 6)**

**Code Changes:**
```python
# config.yaml
features:
  medweb_csv_upload: true
  legacy_excel_upload: false  # REMOVED

# Remove from config:
# - legacy_excel_deprecation_date

# app.py - Remove routes
# DELETE @app.route('/upload-excel')
# DELETE def upload_excel()
# DELETE def parse_roster()
# DELETE def parse_time_range()
# DELETE def backup_dataframe()

# templates/upload.html
# DELETE Excel upload section
# KEEP only medweb CSV section
```

**Deployment:**
1. Final warning email (1 week before)
2. Deploy new version
3. Monitor for issues
4. Update documentation
5. Archive Excel templates

---

## 🚦 Decision Criteria

**Choose Option A (Remove)** if:
- ✅ Single site deployment
- ✅ Full control over migration
- ✅ Medweb CSV is 100% reliable
- ✅ No edge cases requiring manual override
- ✅ Small team, want minimal complexity

**Choose Option B (Hide)** if:
- ✅ Multi-site deployment
- ✅ Need emergency fallback
- ✅ Other departments may adopt
- ✅ Uncertainty about medweb reliability
- ✅ Want maximum flexibility

**Choose Option C (Keep Both)** if:
- ✅ Large diverse user base
- ✅ Mixed workflows required
- ✅ Resources to maintain both paths
- ❌ **NOT recommended** (tech debt)

**Choose Option D (Deprecate)** if:
- ✅ Want clean eventual outcome
- ✅ Can manage 6-month timeline
- ✅ Want usage data to inform decision
- ✅ User-friendly approach
- ✅ **RECOMMENDED** ⭐

---

## 📋 Next Steps

1. **Review this document** with stakeholders
2. **Choose option** (A/B/C/D)
3. **Confirm timeline** (if Option D, set deprecation date)
4. **Implement Phase 1** (medweb CSV upload)
5. **Test thoroughly** with real medweb data
6. **Communicate** to users (if Option D)
7. **Deploy** and monitor

---

## 🤔 Open Questions

1. **Is RadIMO deployed at multiple sites?**
   - Single site → Option A or D
   - Multi-site → Option B or D

2. **How critical is emergency fallback?**
   - Critical (medical) → Option B or D
   - Not critical → Option A or D

3. **Will other departments adopt RadIMO?**
   - Yes → Option B or D
   - No → Option A or D

4. **Team capacity for 6-month deprecation?**
   - Can manage → Option D ⭐
   - Limited capacity → Option B

5. **User base technical sophistication?**
   - High → Option A or D
   - Mixed → Option D
   - Low → Option B (keep fallback)

---

**Document Version:** 1.0
**Date:** 2025-11-20
**Status:** Awaiting Decision

