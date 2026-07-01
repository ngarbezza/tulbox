# Beracode SAST Support Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.
>
> **Desviación de convención:** este repo no usa worktrees, tests automatizados ni bundler (ver `CLAUDE.md`). No hay suite de tests que correr — la verificación de cada tarea es manual, abriendo `beracode/index.html` en el navegador. Se trabaja directo sobre `main`, sin ramas.

**Goal:** Beracode acepta reportes de Veracode Static Analysis (SAST) además de SCA, con una vista de reporte y detalle propia para findings de código.

**Architecture:** Todo vive en `beracode/index.html` (single-file React). Se agrega un parser nuevo (`parseVeracodeStaticJson`), un campo `kind` en el modelo de scan persistido, y dos componentes nuevos (`SastReportView`, `SastDetailDrawer`) que conviven con los existentes (`ReportView`, `DetailDrawer`) sin modificarlos. `App` decide cuál renderizar según `scan.kind`.

**Tech Stack:** React 18 (CDN) + Babel standalone, sin build step. Referencia de diseño: [docs/plans/2026-07-01-beracode-sast-support-design.md](2026-07-01-beracode-sast-support-design.md).

---

### Task 1: Detección de formato, parser SAST y mapeo de severidad

**Files:**
- Modify: `beracode/index.html` (cerca de `parseVeracodeJson`, línea ~565)

**Step 1: Agregar mapeo de severidad numérica y parser SAST**

Justo después de la función `parseVeracodeJson` existente, agregar:

```js
const SAST_SEVERITY_MAP = { 5:'Critical', 4:'High', 3:'Medium', 2:'Low', 1:'Low', 0:'Low' };

function flattenStackFrames(stackDumps) {
  const dumps = stackDumps?.stack_dump || [];
  const frames = [];
  dumps.forEach(d => (d.Frame || []).forEach(f => {
    frames.push({ file: f.SourceFile, line: f.SourceLine, functionName: f.FunctionName });
  }));
  return frames;
}

function parseVeracodeStaticJson(json, filename) {
  try {
    const data = JSON.parse(json);
    const rawFindings = data?.findings || [];
    const matches = rawFindings.map(f => ({
      id: String(f.issue_id),
      severity: SAST_SEVERITY_MAP[f.severity] ?? 'Low',
      issueType: f.issue_type || 'Unknown issue',
      cwe: f.cwe_id || null,
      exploitLevel: f.exploit_level ?? null,
      description: f.display_text || '',
      file: f.files?.source_file?.file || '',
      line: f.files?.source_file?.line ?? null,
      functionName: f.files?.source_file?.function_name || '',
      qualifiedFunctionName: f.files?.source_file?.qualified_function_name || '',
      functionPrototype: f.files?.source_file?.function_prototype || '',
      stackFrames: flattenStackFrames(f.stack_dumps),
      detailsLink: f.flaw_details_link || null,
    }));
    const name = (data?.modules && data.modules[0]) || filename || 'unknown';
    return { id: crypto.randomUUID(), name, scannedAt: new Date().toISOString(), kind: 'sast', matches };
  } catch (e) {
    return null;
  }
}
```

**Step 2: Detectar formato en `DropZone.handleFiles`**

Ubicar la función `handleFiles` dentro de `DropZone` (línea ~705). Reemplazar el bloque que llama a `parseVeracodeJson`:

```js
reader.onload = e => {
  let raw;
  try { raw = JSON.parse(e.target.result); } catch (err) { raw = null; }
  const isSast = raw && Array.isArray(raw.findings) && typeof raw.scan_id === 'string';
  const scan = isSast
    ? parseVeracodeStaticJson(e.target.result, file.name)
    : parseVeracodeJson(e.target.result, file.name);
  if (!scan) { setError(`Could not parse ${file.name}`); return; }
  if (!isSast) scan.kind = 'sca';
  onAdd(scan);
};
```

**Step 3: Marcar `kind: 'sca'` en los seeds existentes**

En `SEED_SCAN_1`, `SEED_SCAN_2`, `SEED_SCAN_3` (líneas ~512-562), agregar `kind: 'sca',` junto a `id:` en cada objeto.

**Step 4: Verificación manual**

Abrir `beracode/index.html` en el navegador. Debe seguir mostrando los 3 seeds SCA sin cambios visuales. Abrir la consola y correr:

```js
parseVeracodeStaticJson(JSON.stringify({scan_id:'x', modules:['foo.jar'], findings:[{issue_id:1, severity:3, issue_type:'Test', files:{source_file:{file:'a.kt', line:1}}}]}), 'test.json')
```

Debe devolver un objeto con `kind: 'sast'` y `matches[0].severity === 'Medium'`.

**Step 5: Commit**

```bash
git add beracode/index.html
git commit -m "feat(beracode): add SAST report parsing and format detection"
```

---

### Task 2: Badge de tipo en `ScanCard`

**Files:**
- Modify: `beracode/index.html:751-817` (`ScanCard`)

**Step 1: Agregar badge y label condicional del pie**

En el header row de `ScanCard` (línea ~778, dentro del `<div style={{flex:1,...}}>`), agregar el badge de tipo justo debajo del nombre:

```jsx
<div style={{ fontSize:14, fontWeight:600, color:'var(--text)', overflow:'hidden', textOverflow:'ellipsis', whiteSpace:'nowrap', fontFamily:'var(--font)' }}>{scan.name}</div>
<div style={{ display:'flex', alignItems:'center', gap:6, marginTop:3 }}>
  <span style={{ fontSize:10, fontWeight:600, letterSpacing:'0.04em', color:'var(--text-faint)', border:'1px solid var(--border)', borderRadius:4, padding:'1px 5px' }}>
    {scan.kind === 'sast' ? 'SAST' : 'SCA'}
  </span>
  <span style={{ fontSize:11, color:'var(--text-faint)' }}>{formatDate(scan.scannedAt)}</span>
</div>
<div style={{ fontSize:11, color:'var(--text-faint)', marginTop:1 }}>{relativeTime(scan.scannedAt)}</div>
```

(Esto reemplaza el bloque de fecha existente, incorporando el badge en la misma fila.)

**Step 2: Label del pie según tipo**

En el footer de `ScanCard` (línea ~811), reemplazar:

```jsx
<span style={{ fontSize:11, color:'var(--text-faint)' }}>
  {scan.kind === 'sast'
    ? `${new Set(scan.matches.map(v=>v.issueType)).size} issue type${new Set(scan.matches.map(v=>v.issueType)).size!==1?'s':''}`
    : `${s.pkgs} package${s.pkgs!==1?'s':''} affected`}
</span>
```

**Step 3: Verificación manual**

Recargar `index.html`. Los 3 seeds SCA deben mostrar badge "SCA" y "N packages affected" sin cambios de layout.

**Step 4: Commit**

```bash
git add beracode/index.html
git commit -m "feat(beracode): show scan type badge on scan cards"
```

---

### Task 3: `SastReportView`

**Files:**
- Modify: `beracode/index.html` (agregar componente nuevo cerca de `ReportView`, línea ~1092, antes de `Pill`)

**Step 1: Crear el componente**

Reutiliza `SEV_ORDER`, `sevColor`, `sevBg`, `scanSummary`, `SevBadge`, `Logo`, `Pill`, `formatDate` ya definidos. Estructura calcada de `ReportView` pero con `Issue Type` en vez de `Package` y columnas de tabla propias:

```jsx
function SastReportView({ scan, onBack }) {
  const [sevFilter, setSevFilter]   = useState(null);
  const [typeFilter, setTypeFilter] = useState(null);
  const [search, setSearch]         = useState('');
  const [selected, setSelected]     = useState(null);

  const types = useMemo(() => {
    const map = {};
    scan.matches.forEach(v => {
      if (!map[v.issueType]) map[v.issueType] = [];
      map[v.issueType].push(v);
    });
    return map;
  }, [scan]);

  const summary = useMemo(() => scanSummary(scan.matches), [scan]);

  const filtered = useMemo(() => {
    let data = scan.matches;
    if (sevFilter) data = data.filter(v=>v.severity===sevFilter);
    if (typeFilter) data = data.filter(v=>v.issueType===typeFilter);
    if (search.trim()) {
      const q = search.trim().toLowerCase();
      data = data.filter(v =>
        v.issueType.toLowerCase().includes(q) ||
        (v.cwe && v.cwe.toLowerCase().includes(q)) ||
        v.file.toLowerCase().includes(q) ||
        v.functionName.toLowerCase().includes(q));
    }
    return [...data].sort((a,b) => (SEV_ORDER[a.severity]??9)-(SEV_ORDER[b.severity]??9));
  }, [scan, sevFilter, typeFilter, search]);

  return (
    <div style={{ height:'100vh', display:'flex', flexDirection:'column', background:'var(--bg)' }}>
      <div style={{ borderBottom:'1px solid var(--border)', padding:'0 24px', display:'flex', alignItems:'center', gap:12, height:52, flexShrink:0 }}>
        <button onClick={onBack} style={{ display:'flex', alignItems:'center', gap:6, background:'none', border:'none', color:'var(--text-muted)', cursor:'pointer', fontSize:13, padding:'4px 8px', borderRadius:6 }}>
          <span style={{ fontSize:16 }}>←</span> All scans
        </button>
        <div style={{ width:1, height:20, background:'var(--border)' }}/>
        <Logo/>
        <div style={{ marginLeft:'auto', textAlign:'right' }}>
          <div style={{ fontSize:13, fontWeight:600, color:'var(--text)', fontFamily:'var(--font)' }}>{scan.name}</div>
          <div style={{ fontSize:11, color:'var(--text-faint)' }}>{formatDate(scan.scannedAt)}</div>
        </div>
      </div>

      <div style={{ flex:1, display:'grid', gridTemplateColumns:'244px 1fr', overflow:'hidden' }}>
        <div style={{ borderRight:'1px solid var(--border)', overflowY:'auto', padding:'16px 12px', display:'flex', flexDirection:'column', gap:20 }}>
          <input value={search} onChange={e=>setSearch(e.target.value)} placeholder="Search issue, CWE, file…"
            style={{ width:'100%', padding:'7px 9px', background:'var(--surface2)', border:'1px solid var(--border)', borderRadius:6, color:'var(--text)', fontSize:12, outline:'none' }}/>

          <div>
            <div style={{ fontSize:10, fontWeight:600, color:'var(--text-faint)', letterSpacing:'0.1em', textTransform:'uppercase', marginBottom:8 }}>Severity</div>
            {['Critical','High','Medium','Low'].map(s => {
              const count = scan.matches.filter(v=>v.severity===s).length;
              const active = sevFilter===s;
              return (
                <button key={s} onClick={()=>setSevFilter(active?null:s)} style={{ display:'flex', alignItems:'center', justifyContent:'space-between', width:'100%', padding:'5px 8px', borderRadius:6, border:'none', cursor:'pointer', background:active?sevBg(s):'transparent', textAlign:'left', marginBottom:2 }}>
                  <div style={{ display:'flex', alignItems:'center', gap:7 }}>
                    <div style={{ width:7, height:7, borderRadius:'50%', background:sevColor(s) }}/>
                    <span style={{ fontSize:12, color:active?sevColor(s):'var(--text)', fontWeight:active?600:400 }}>{s}</span>
                  </div>
                  <span style={{ fontSize:11, color:'var(--text-faint)', fontFamily:'var(--font)' }}>{count}</span>
                </button>
              );
            })}
          </div>

          <div>
            <div style={{ fontSize:10, fontWeight:600, color:'var(--text-faint)', letterSpacing:'0.1em', textTransform:'uppercase', marginBottom:8 }}>Issue type</div>
            {Object.entries(types).map(([name, vulns]) => {
              const active = typeFilter===name;
              const topSev = ['Critical','High','Medium','Low'].find(s=>vulns.some(v=>v.severity===s));
              return (
                <button key={name} onClick={()=>setTypeFilter(active?null:name)} style={{ display:'grid', gridTemplateColumns:'1fr auto auto', gap:8, alignItems:'center', padding:'7px 8px', background:active?'var(--surface2)':'transparent', border:`1px solid ${active?'var(--border)':'transparent'}`, borderRadius:6, cursor:'pointer', textAlign:'left', width:'100%', marginBottom:2 }}>
                  <div style={{ fontSize:12, fontWeight:500, color:'var(--text)', overflow:'hidden', textOverflow:'ellipsis', whiteSpace:'nowrap' }}>{name}</div>
                  {topSev && <SevBadge sev={topSev} small/>}
                  <span style={{ fontSize:11, fontFamily:'var(--font)', color:'var(--text-faint)' }}>{vulns.length}</span>
                </button>
              );
            })}
          </div>
        </div>

        <div style={{ overflowY:'auto', display:'flex', flexDirection:'column' }}>
          <div style={{ padding:'16px 20px', borderBottom:'1px solid var(--border)', display:'grid', gridTemplateColumns:'repeat(5,1fr)', gap:10 }}>
            {[
              { label:'Total', value:summary.total, color:'var(--text)', f:null },
              { label:'Critical', value:summary.Critical, color:'var(--critical)', f:'Critical' },
              { label:'High', value:summary.High, color:'var(--high)', f:'High' },
              { label:'Medium', value:summary.Medium, color:'var(--medium)', f:'Medium' },
              { label:'Low', value:summary.Low, color:'var(--low)', f:'Low' },
            ].map(c => {
              const active = sevFilter===c.f;
              return (
                <button key={c.label} onClick={()=>setSevFilter(active?null:c.f)} style={{ background:active?(c.f?sevBg(c.f):'var(--surface2)'):'var(--surface)', border:`1px solid ${active?(c.f?sevColor(c.f)+'55':'var(--accent)'):'var(--border)'}`, borderRadius:8, padding:'12px 16px', cursor:'pointer', textAlign:'left' }}>
                  <div style={{ fontSize:28, fontWeight:700, color:c.color, fontFamily:'var(--font)', lineHeight:1 }}>{c.value}</div>
                  <div style={{ fontSize:12, fontWeight:600, color:'var(--text)', marginTop:3 }}>{c.label}</div>
                </button>
              );
            })}
          </div>

          {(sevFilter||typeFilter||search) && (
            <div style={{ padding:'8px 20px', display:'flex', alignItems:'center', gap:8, borderBottom:'1px solid var(--border-subtle)', flexWrap:'wrap' }}>
              <span style={{ fontSize:11, color:'var(--text-faint)' }}>Filters:</span>
              {sevFilter && <Pill label={sevFilter} color={sevColor(sevFilter)} onRemove={()=>setSevFilter(null)}/>}
              {typeFilter && <Pill label={typeFilter} onRemove={()=>setTypeFilter(null)}/>}
              {search && <Pill label={`"${search}"`} onRemove={()=>setSearch('')}/>}
              <button onClick={()=>{setSevFilter(null);setTypeFilter(null);setSearch('');}} style={{ fontSize:11, color:'var(--text-faint)', background:'none', border:'none', cursor:'pointer' }}>Clear all</button>
              <span style={{ fontSize:11, color:'var(--text-faint)', marginLeft:'auto' }}>{filtered.length} finding{filtered.length!==1?'s':''}</span>
            </div>
          )}

          <div style={{ flex:1 }}>
            {filtered.length===0 ? (
              <div style={{ padding:60, textAlign:'center', color:'var(--text-faint)' }}>
                <div style={{ fontSize:28, marginBottom:10 }}>✓</div>
                <div style={{ fontSize:13 }}>No findings match your filters</div>
              </div>
            ) : (
              <table style={{ width:'100%', borderCollapse:'collapse', fontSize:13 }}>
                <thead>
                  <tr style={{ borderBottom:'1px solid var(--border)' }}>
                    {['Severity','Issue','CWE','Location','Exploit'].map(l=>(
                      <th key={l} style={{ padding:'8px 14px', textAlign:'left', fontSize:10, fontWeight:600, letterSpacing:'0.06em', textTransform:'uppercase', color:'var(--text-faint)', background:'var(--surface)', position:'sticky', top:0, borderBottom:'1px solid var(--border)', whiteSpace:'nowrap' }}>{l}</th>
                    ))}
                  </tr>
                </thead>
                <tbody>
                  {filtered.map((v,i)=>{
                    const active = selected && selected.id===v.id;
                    return (
                      <tr key={v.id+i} onClick={()=>setSelected(active?null:v)} style={{ cursor:'pointer', background:active?'var(--surface2)':'transparent', borderBottom:'1px solid var(--border-subtle)' }}>
                        <td style={{ padding:'9px 14px' }}><SevBadge sev={v.severity} small/></td>
                        <td style={{ padding:'9px 14px', maxWidth:280 }}><div style={{ fontSize:12, color:'var(--text)', lineHeight:1.4 }}>{v.issueType}</div></td>
                        <td style={{ padding:'9px 14px' }}><span style={{ fontSize:11, fontFamily:'var(--font)', color:'var(--text-muted)' }}>{v.cwe || '—'}</span></td>
                        <td style={{ padding:'9px 14px' }}>
                          <div style={{ fontSize:11, fontFamily:'var(--font)', color:'var(--accent)' }}>{v.file}:{v.line}</div>
                          <div style={{ fontSize:10, color:'var(--text-faint)', fontFamily:'var(--font)' }}>{v.functionName}()</div>
                        </td>
                        <td style={{ padding:'9px 14px' }}><span style={{ fontSize:11, fontFamily:'var(--font)', color:'var(--text-muted)' }}>{v.exploitLevel ?? '—'}</span></td>
                      </tr>
                    );
                  })}
                </tbody>
              </table>
            )}
          </div>
        </div>
      </div>

      {selected && <SastDetailDrawer vuln={selected} onClose={()=>setSelected(null)}/>}
    </div>
  );
}
```

**Step 2: Verificación manual**

Se completa junto con el Task 4 (necesita seeds/datos para probarse visualmente) — pasar a Task 4 antes de verificar.

**Step 3: Commit**

Se commitea junto con el Task 4 para tener algo probable en pantalla.

---

### Task 4: `SastDetailDrawer`

**Files:**
- Modify: `beracode/index.html` (agregar cerca de `DetailDrawer`, línea ~873, antes de `ReportView`)

**Step 1: Crear el componente**

Reutiliza `DSection`, `KV`, `SevBadge`. Sanitiza `display_text` con una whitelist mínima de tags (usa `dangerouslySetInnerHTML` solo porque el contenido viene de Veracode, no de input de usuario, y ya lo hacía implícitamente el reporte HTML de Veracode — se limpia igual con una regex simple para evitar tags fuera de la whitelist):

```jsx
function sanitizeVeracodeHtml(html) {
  const allowed = ['span', 'a', 'br'];
  return html.replace(/<(\/?)(\w+)([^>]*)>/g, (m, close, tag, attrs) => {
    if (!allowed.includes(tag.toLowerCase())) return '';
    if (tag.toLowerCase() === 'a' && !close) {
      const href = attrs.match(/href="([^"]*)"/);
      return href ? `<a href="${href[1]}" target="_blank" rel="noopener">` : '<a>';
    }
    return `<${close}${tag}>`;
  });
}

function SastDetailDrawer({ vuln, onClose }) {
  useEffect(() => {
    const h = e => { if(e.key==='Escape') onClose(); };
    window.addEventListener('keydown', h);
    return () => window.removeEventListener('keydown', h);
  }, [onClose]);
  if (!vuln) return null;
  return (
    <>
      <div onClick={onClose} style={{ position:'fixed', inset:0, background:'rgba(0,0,0,0.45)', zIndex:100, backdropFilter:'blur(2px)' }}/>
      <div style={{ position:'fixed', right:0, top:0, bottom:0, width:460, background:'var(--surface)', borderLeft:'1px solid var(--border)', zIndex:101, display:'flex', flexDirection:'column', overflowY:'auto' }}>
        <div style={{ padding:'20px 24px', borderBottom:'1px solid var(--border)', display:'flex', alignItems:'flex-start', gap:12 }}>
          <div style={{ flex:1 }}>
            <div style={{ display:'flex', alignItems:'center', gap:10, flexWrap:'wrap' }}>
              <SevBadge sev={vuln.severity}/>
              {vuln.cwe && <span style={{ fontSize:12, color:'var(--text-muted)', fontFamily:'var(--font)' }}>CWE-{vuln.cwe}</span>}
            </div>
            <div style={{ fontSize:14, fontWeight:600, color:'var(--text)', marginTop:8 }}>{vuln.issueType}</div>
          </div>
          <button onClick={onClose} style={{ background:'none', border:'none', color:'var(--text-muted)', cursor:'pointer', fontSize:20, lineHeight:1, padding:'0 4px', borderRadius:4 }}>×</button>
        </div>
        <div style={{ padding:'20px 24px', flex:1, display:'flex', flexDirection:'column', gap:20 }}>
          <div style={{ fontSize:13, color:'var(--text-muted)', lineHeight:1.7 }}
               dangerouslySetInnerHTML={{ __html: sanitizeVeracodeHtml(vuln.description) }}/>

          <DSection label="Location">
            <KV label="File" value={<span className="mono" style={{color:'var(--accent)'}}>{vuln.file}:{vuln.line}</span>}/>
            <KV label="Function" value={<span className="mono">{vuln.qualifiedFunctionName || vuln.functionName}</span>}/>
            {vuln.functionPrototype && <KV label="Prototype" value={<span className="mono" style={{fontSize:11}}>{vuln.functionPrototype}</span>}/>}
          </DSection>

          {vuln.stackFrames.length > 0 && (
            <DSection label="Data flow">
              <div style={{ display:'flex', flexDirection:'column', gap:6 }}>
                {vuln.stackFrames.map((f,i) => (
                  <div key={i} style={{ fontSize:12, fontFamily:'var(--font)', color:'var(--text-muted)' }}>
                    {f.file}:{f.line} — <span style={{color:'var(--text)'}}>{f.functionName}()</span>
                  </div>
                ))}
              </div>
            </DSection>
          )}

          {vuln.detailsLink && (
            <DSection label="References">
              <a href={vuln.detailsLink} target="_blank" rel="noopener" style={{ fontSize:12, color:'var(--accent)', fontFamily:'var(--font)', wordBreak:'break-all', textDecoration:'none' }}>CWE-{vuln.cwe} details</a>
            </DSection>
          )}
        </div>
      </div>
    </>
  );
}
```

**Step 2: Dispatch en `App` según `scan.kind`**

En `App` (línea ~1209), reemplazar:

```jsx
{activeScan
  ? (activeScan.kind === 'sast'
      ? <SastReportView scan={activeScan} onBack={()=>setActiveScan(null)}/>
      : <ReportView scan={activeScan} onBack={()=>setActiveScan(null)}/>)
  : <HomeView scans={scans} onOpen={setActiveScan} onDelete={deleteScan} onAdd={addScan}/>
}
```

**Step 3: Verificación manual**

Abrir `index.html`, soltar el JSON de ejemplo real (`/Users/nahuel/veracode-reports/static/becom-catalog/2026-07-01-8ec2519.json`) en el drop zone. Verificar:
- Se agrega una card con badge "SAST" y "1 issue type".
- Al abrir, la tabla muestra 1 fila: severidad Medium, issue "Improper Output Neutralization for Logs", CWE 117, `com/claro/.../SapStockClient.kt:53`, función `processErrors`.
- Al clickear la fila, el drawer muestra la descripción, ubicación, data flow (2 frames) y el link a CWE-117.

También soltar `/Users/nahuel/veracode-reports/static/ab-testing-engine/2026-07-01-5850e39.json` (0 findings) y verificar que se agregue una card con badge SAST, 0 en todas las severidades, y que al abrirla la tabla muestre "No findings match your filters" sin romper nada (findings vacío es un caso válido, no un error de parseo).

**Step 4: Commit**

```bash
git add beracode/index.html
git commit -m "feat(beracode): add SAST report view and finding detail drawer"
```

---

### Task 5: Seeds de demo SAST y copy del DropZone

**Files:**
- Modify: `beracode/index.html` (seeds cerca de línea ~562, `DropZone` copy línea ~744)

**Step 1: Agregar seed SAST**

Después de `SEED_SCAN_3`, agregar (nombres y hallazgos ficticios, variedad de severidades/CWE):

```js
const SEED_SCAN_SAST_1 = {
  id: 'seed-sast-1',
  name: '[DEMO] legacy-billing-service-3.2.0.jar',
  scannedAt: '2025-05-10T10:00:00-03:00',
  kind: 'sast',
  matches: [
    { id:'2001', severity:'Critical', issueType:'SQL Injection', cwe:'89', exploitLevel:1, description:'<span>Tainted data flows into a SQL query without sanitization, allowing SQL injection.</span> <span>Use parameterized queries or prepared statements.</span>', file:'com/acme/billing/InvoiceDao.java', line:88, functionName:'findByCustomer', qualifiedFunctionName:'com.acme.billing.InvoiceDao.findByCustomer', functionPrototype:'List findByCustomer(String)', stackFrames:[{file:'com/acme/billing/InvoiceDao.java',line:88,functionName:'findByCustomer'},{file:'com/acme/billing/InvoiceController.java',line:41,functionName:'search'}], detailsLink:'https://downloads.veracode.com/securityscan/cwe/v4/java/89.html' },
    { id:'2002', severity:'High', issueType:'Hardcoded Credentials', cwe:'798', exploitLevel:1, description:'<span>A hardcoded password is used to authenticate against the payment gateway.</span> <span>Store credentials in a secrets manager instead.</span>', file:'com/acme/billing/gateway/PaymentClient.java', line:22, functionName:'connect', qualifiedFunctionName:'com.acme.billing.gateway.PaymentClient.connect', functionPrototype:'Connection connect()', stackFrames:[], detailsLink:'https://downloads.veracode.com/securityscan/cwe/v4/java/798.html' },
    { id:'2003', severity:'High', issueType:'XML External Entity (XXE)', cwe:'611', exploitLevel:1, description:'<span>The XML parser resolves external entities, enabling XXE attacks on uploaded invoices.</span>', file:'com/acme/billing/InvoiceImporter.java', line:57, functionName:'parse', qualifiedFunctionName:'com.acme.billing.InvoiceImporter.parse', functionPrototype:'Invoice parse(InputStream)', stackFrames:[{file:'com/acme/billing/InvoiceImporter.java',line:57,functionName:'parse'}], detailsLink:'https://downloads.veracode.com/securityscan/cwe/v4/java/611.html' },
    { id:'2004', severity:'Medium', issueType:'Improper Output Neutralization for Logs', cwe:'117', exploitLevel:1, description:'<span>Untrusted data is written to logs without sanitization, enabling log forging.</span>', file:'com/acme/billing/AuditLogger.java', line:33, functionName:'logAttempt', qualifiedFunctionName:'com.acme.billing.AuditLogger.logAttempt', functionPrototype:'void logAttempt(String)', stackFrames:[], detailsLink:'https://downloads.veracode.com/securityscan/cwe/v4/java/117.html' },
    { id:'2005', severity:'Medium', issueType:'Insecure Randomness', cwe:'330', exploitLevel:0, description:'<span>A non-cryptographic random generator is used to create invoice tokens.</span>', file:'com/acme/billing/TokenFactory.java', line:14, functionName:'newToken', qualifiedFunctionName:'com.acme.billing.TokenFactory.newToken', functionPrototype:'String newToken()', stackFrames:[], detailsLink:null },
    { id:'2006', severity:'Low', issueType:'Information Exposure', cwe:null, exploitLevel:0, description:'<span>A verbose error message exposes the internal stack trace to API clients.</span>', file:'com/acme/billing/ApiErrorHandler.java', line:19, functionName:'handle', qualifiedFunctionName:'com.acme.billing.ApiErrorHandler.handle', functionPrototype:'Response handle(Exception)', stackFrames:[], detailsLink:null },
  ],
};
```

**Step 2: Agregar el seed al estado inicial de `App`**

En `App`, línea ~1173, cambiar el fallback:

```js
return [SEED_SCAN_1, SEED_SCAN_2, SEED_SCAN_3, SEED_SCAN_SAST_1];
```

**Step 3: Actualizar copy del `DropZone`**

Línea ~744, cambiar:

```jsx
<div style={{ fontSize:12, color:'var(--text-faint)' }}>or click to browse · SCA or static analysis result.json</div>
```

**Step 4: Verificación manual**

Borrar `localStorage` (`localStorage.removeItem('beracode_scans')` en consola) y recargar. Deben aparecer 4 cards: 3 SCA + 1 SAST con badge y datos correctos. Abrir el seed SAST y verificar que la tabla y el drawer funcionen igual que con el JSON real del Task 4.

**Step 5: Commit y push**

```bash
git add beracode/index.html docs/plans/2026-07-01-beracode-sast-support-design.md docs/plans/2026-07-01-beracode-sast-support-plan.md
git commit -m "feat(beracode): add SAST demo seed and update drop zone copy"
git push origin main
```
