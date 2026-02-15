<%*
// --- 1. ZENTRALES MAPPING AUS .MD DATEI LADEN ---
const MAPPING_FILE_NAME = "field-mapping"; 
let fieldMapping = {};

try {
    const tFile = tp.file.find_tfile(MAPPING_FILE_NAME);
    if (tFile) {
        const content = await app.vault.read(tFile);
        const lines = content.split("\n");
        lines.forEach(line => {
            const parts = line.split(":");
            if (parts.length === 2) {
                fieldMapping[parts[0].trim()] = parts[1].trim();
            }
        });
    }
} catch (e) {
    console.log("Keine Mapping-Datei gefunden.");
}

// --- 2. CSS FÜR DIE UI ---
const styleId = 'gnd-style-sheet';
if (!document.getElementById(styleId)) {
    const style = document.createElement('style');
    style.id = styleId;
    style.innerHTML = `
        .gnd-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.7); z-index: 10000; display: flex; justify-content: center; align-items: flex-start; padding-top: 50px; font-family: var(--font-interface); }
        .gnd-modal { background: var(--background-primary); width: 750px; border-radius: 10px; box-shadow: var(--shadow-l); border: 1px solid var(--background-modifier-border); overflow: hidden; display: flex; flex-direction: column; max-height: 85vh; color: var(--text-normal); }
        .gnd-search-bar { display: flex; padding: 15px; border-bottom: 1px solid var(--background-modifier-border); gap: 12px; align-items: center; background: var(--background-secondary); }
        .gnd-input { flex-grow: 1; background: var(--background-primary); border: 1px solid var(--background-modifier-border); border-radius: 6px; padding: 10px 15px; color: var(--text-normal); font-size: 1.1em; outline: none; }
        .gnd-results, .gnd-preview-list { overflow-y: auto; background: var(--background-primary); flex-grow: 1; }
        .gnd-item { padding: 15px; cursor: pointer; border-bottom: 1px solid var(--background-modifier-border); transition: background 0.1s; }
        .gnd-item:hover { background: var(--background-modifier-hover); }
        .gnd-header-row { display: flex; justify-content: space-between; align-items: center; width: 100%; gap: 10px; }
        .gnd-item-title { font-weight: 600; color: var(--text-accent); font-size: 1.05em; flex-grow: 1; }
        .gnd-badge { font-size: 0.65em; padding: 2px 8px; border-radius: 4px; text-transform: uppercase; font-weight: bold; background: var(--background-modifier-border); color: var(--text-muted); border: 1px solid var(--background-modifier-border); }
        .gnd-preview-item { display: flex; align-items: flex-start; padding: 10px 15px; gap: 12px; border-bottom: 1px solid var(--background-modifier-border); }
        .gnd-preview-key { font-weight: bold; color: var(--text-accent); width: 220px; flex-shrink: 0; cursor: pointer; }
        .gnd-preview-val { color: var(--text-muted); font-size: 0.9em; line-height: 1.4; }
        .gnd-footer { padding: 15px; background: var(--background-secondary); display: flex; justify-content: flex-end; gap: 10px; border-top: 1px solid var(--background-modifier-border); }
        .gnd-btn { padding: 8px 18px; border-radius: 5px; cursor: pointer; border: none; font-weight: 600; }
        .gnd-btn-primary { background: var(--interactive-accent); color: var(--text-on-accent); }
        .gnd-btn-secondary { background: var(--background-modifier-border); color: var(--text-normal); }
        .badge-Person { background-color: #2d4f1e; color: #a3e635; }
        .badge-SubjectHeading { background-color: #1e3a8a; color: #93c5fd; }
    `;
    document.head.appendChild(style);
}

function formatType(types) {
    if (!types) return { l: "GND", c: "" };
    const tArray = Array.isArray(types) ? types : [types];
    const rawType = tArray.filter(t => t !== "AuthorityResource")[0] || tArray[0];
    return { l: rawType.replace(/([A-Z])/g, ' $1').trim(), c: "badge-" + rawType };
}

function harvest(val) {
    if (Array.isArray(val)) {
        const list = val.map(harvest);
        return list.length === 1 ? list[0] : list;
    }
    if (val && typeof val === 'object') return val.label || val.id || JSON.stringify(val);
    return val;
}

// --- 3. MODAL FLOW ---
const resultData = await new Promise((resolve) => {
    const overlay = document.createElement('div');
    overlay.className = 'gnd-overlay';
    overlay.innerHTML = `<div class="gnd-modal">
        <div class="gnd-search-bar">
            <input type="text" class="gnd-input" placeholder="GND Live-Suche..." spellcheck="false">
            <button class="gnd-close" style="background:none; border:none; cursor:pointer; color:var(--text-muted); font-size: 24px;">✕</button>
        </div>
        <div class="gnd-results"><div style="padding:40px; text-align:center; color:var(--text-muted);">Suchbegriff eingeben...</div></div>
    </div>`;
    document.body.appendChild(overlay);

    const input = overlay.querySelector('.gnd-input');
    const resultsDiv = overlay.querySelector('.gnd-results');
    let debounceTimer;
    input.focus();

    const cleanup = () => { if(overlay.parentNode) document.body.removeChild(overlay); };
    overlay.querySelector('.gnd-close').onclick = () => { cleanup(); resolve(null); };
    
    input.addEventListener('input', () => {
        const query = input.value.trim();
        clearTimeout(debounceTimer);
        if (query.length < 2) return;
        debounceTimer = setTimeout(async () => {
            try {
                const res = await requestUrl({ url: `https://lobid.org/gnd/search?q=${encodeURIComponent(query)}&format=json&size=15` });
                const members = res.json.member;
                resultsDiv.innerHTML = '';
                if (!members) return;
                members.forEach(item => {
                    const info = formatType(item.type);
                    const el = document.createElement('div');
                    el.className = 'gnd-item';
                    el.innerHTML = `<div class="gnd-header-row"><span class="gnd-item-title">${item.preferredName}</span><span class="gnd-badge ${info.c}">${info.l}</span></div>`;
                    el.onclick = async () => {
                        resultsDiv.innerHTML = '<div style="padding:40px; text-align:center;">Lade Datensatz...</div>';
                        const fullData = (await requestUrl({ url: `https://lobid.org/gnd/${item.gndIdentifier}.json` })).json;
                        showPreview(fullData, overlay, resolve, cleanup);
                    };
                    resultsDiv.appendChild(el);
                });
            } catch (err) { resultsDiv.innerHTML = 'Fehler.'; }
        }, 400);
    });

    function showPreview(data, overlay, resolve, cleanup) {
        const blackList = ["variantNameEntityForThePerson", "context"];
        let previewItems = [];
        
        for (const [key, value] of Object.entries(data)) {
            if (blackList.includes(key)) continue;
            
            // LOGIK: Ist der Key im Mapping?
            const isMapped = fieldMapping.hasOwnProperty(key);
            const finalKey = isMapped ? fieldMapping[key] : key.replace(/[^a-zA-Z0-9_]/g, "_").replace(/^_+|_+$/g, "");
            
            previewItems.push({ 
                key: finalKey, 
                value: harvest(value),
                autoCheck: isMapped // Nur markieren, wenn im Mapping gefunden
            });
        }

        const modal = overlay.querySelector('.gnd-modal');
        modal.innerHTML = `
            <div class="gnd-search-bar"><h3>Vorschau: ${data.preferredName}</h3></div>
            <div class="gnd-preview-list">
                ${previewItems.map((item, i) => `
                    <div class="gnd-preview-item" style="${!item.autoCheck ? 'opacity: 0.7;' : ''}">
                        <input type="checkbox" id="gnd-c-${i}" ${item.autoCheck ? 'checked' : ''}>
                        <label for="gnd-c-${i}" class="gnd-preview-key">${item.key}</label>
                        <div class="gnd-preview-val">${item.value}</div>
                    </div>
                `).join('')}
            </div>
            <div class="gnd-footer">
                <button class="gnd-btn gnd-btn-secondary" id="gnd-back">Zurück</button>
                <button class="gnd-btn gnd-btn-primary" id="gnd-import">Importieren</button>
            </div>
        `;

        modal.querySelector('#gnd-back').onclick = () => { cleanup(); resolve('RESTART'); };
        modal.querySelector('#gnd-import').onclick = () => {
            let finalFields = {};
            previewItems.forEach((item, i) => {
                if (modal.querySelector(`#gnd-c-${i}`).checked) finalFields[item.key] = item.value;
            });
            cleanup();
            resolve({ fields: finalFields, original: data });
        };
    }
});

if (!resultData) return;
if (resultData === 'RESTART') { tp.file.execute_active_template(); return; }

const { fields, original } = resultData;

// --- 4. DATEI AKTUALISIEREN ---
const activeFile = app.workspace.getActiveFile();
if (activeFile) {
    await app.fileManager.processFrontMatter(activeFile, (fm) => {
        for (const [k, v] of Object.entries(fields)) fm[k] = v;
    });

    const titleHeading = `# ${original.preferredName}`;
    
    // --- NEU: Depiction / Bild-Logik ---
    let imageMarkdown = "";
    if (fields.depiction) {
        // Falls es mehrere Bilder sind, nehmen wir das erste, ansonsten direkt den String
        const imgUrl = Array.isArray(fields.depiction) ? fields.depiction[0] : fields.depiction;
        imageMarkdown = `\n\n![Portrait](${imgUrl})`;
    }

    const bioText = original.biographicalOrHistoricalInformation ? 
        (Array.isArray(original.biographicalOrHistoricalInformation) ? original.biographicalOrHistoricalInformation.join("\n\n") : original.biographicalOrHistoricalInformation) : "";
    
    let links = [`- [DNB Katalog](https://d-nb.info/gnd/${original.gndIdentifier})`, `- [GND Explorer](https://lobid.org/gnd/${original.gndIdentifier})` ];
    if (original.sameAs) original.sameAs.forEach(s => {
        const label = (typeof s === 'object') ? (s.collection?.abbr || s.collection?.name || "Link") : "Link";
        const url = (typeof s === 'object') ? s.id : s;
        links.push(`- [${label}](${url})`);
    });
    const footer = `\n\n## GND Links\n${links.join("\n")}`;

    await app.vault.process(activeFile, (content) => {
        let newContent = content;
        // Das Bild wird jetzt direkt unter die Überschrift gesetzt
        const bodyHeader = titleHeading + imageMarkdown + (bioText ? "\n\n" + bioText : "");
        
        if (newContent.startsWith("---")) {
            const endOfFM = newContent.indexOf("---", 3);
            if (endOfFM !== -1) {
                const fmBlock = newContent.substring(0, endOfFM + 3);
                const rest = newContent.substring(endOfFM + 3).trim();
                if (!rest.includes(titleHeading)) {
                    newContent = `${fmBlock}\n\n${bodyHeader}\n\n${rest}`;
                }
            }
        } else if (!newContent.trim().startsWith("# ")) {
            newContent = bodyHeader + "\n\n" + newContent.trim();
        }
        
        if (!newContent.includes("## GND Links")) {
            newContent = newContent.trim() + footer;
        }
        return newContent;
    });
    new Notice("GND Import abgeschlossen inkl. Bild!");
}
%>