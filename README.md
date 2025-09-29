<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Instant Privacy Tool — Redact Sensitive Info</title>
<style>
  :root{--bg:#0f1724;--card:#0b1220;--muted:#9aa7bd;--accent:#6ee7b7}
  body{margin:0;font-family:Inter,system-ui,Segoe UI,Roboto,"Helvetica Neue",Arial;background:linear-gradient(180deg,#071028 0%, #081421 100%);color:#e6eef7;min-height:100vh;display:flex;align-items:center;justify-content:center;padding:28px}
  .wrap{width:100%;max-width:980px;background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));border-radius:12px;padding:22px;box-shadow:0 8px 30px rgba(2,6,23,0.7)}
  h1{margin:0 0 8px;font-size:20px}
  p.lead{margin:0 0 18px;color:var(--muted)}
  .grid{display:grid;grid-template-columns:1fr 360px;gap:18px}
  textarea{width:100%;min-height:260px;padding:14px;border-radius:8px;border:1px solid rgba(255,255,255,0.04);background:rgba(255,255,255,0.02);color:inherit;resize:vertical;font-size:14px}
  .controls{background:rgba(255,255,255,0.01);border-radius:8px;padding:12px;border:1px solid rgba(255,255,255,0.02)}
  label.option{display:flex;align-items:center;gap:10px;margin-bottom:8px}
  input[type="checkbox"]{width:18px;height:18px}
  .btn{display:inline-block;background:linear-gradient(90deg,#10b981,#06b6d4);padding:10px 14px;border-radius:8px;color:#042027;text-weight:600;border:none;cursor:pointer;margin-right:8px}
  .btn-secondary{background:transparent;border:1px solid rgba(255,255,255,0.06);color:var(--muted)}
  .small{font-size:13px;color:var(--muted)}
  .outActions{display:flex;gap:8px;align-items:center;margin-top:8px}
  .footer{margin-top:14px;font-size:13px;color:var(--muted)}
  .pill{display:inline-block;background:rgba(255,255,255,0.03);padding:6px 8px;border-radius:999px;font-size:13px;color:var(--muted)}
  .topRow{display:flex;gap:10px;align-items:center;justify-content:space-between;margin-bottom:10px}
  .notice{background:rgba(255,255,255,0.02);padding:8px;border-radius:8px;color:var(--muted);font-size:13px}
  .tag{background:rgba(255,255,255,0.03);padding:6px 8px;border-radius:6px;font-size:12px;color:var(--muted);display:inline-block}
  .flexCol{display:flex;flex-direction:column}
  .sample{font-size:13px;color:var(--muted);margin-bottom:8px}
  @media(max-width:880px){.grid{grid-template-columns:1fr;}.wrap{padding:16px}}
</style>
</head>
<body>
  <div class="wrap" role="main">
    <div class="topRow">
      <div>
        <h1>Instant Privacy Tool</h1>
        <p class="lead">Paste text → choose what to redact → click <strong>Sanitize</strong>. All done locally in your browser.</p>
      </div>
      <div style="text-align:right">
        <span class="pill">Client-side</span>
      </div>
    </div>

    <div class="grid">
      <div>
        <div class="sample small">Input text (private — stays in your browser):</div>
        <textarea id="inputText" placeholder="Paste resume, message, post, or any text here..."></textarea>

        <div style="margin-top:10px;display:flex;gap:8px;flex-wrap:wrap;align-items:center">
          <button id="sanitizeBtn" class="btn">Sanitize →</button>
          <button id="clearBtn" class="btn-secondary">Clear</button>
          <div class="small" style="margin-left:auto">Characters: <span id="charCount">0</span></div>
        </div>

        <div style="margin-top:12px" class="notice">
          Tip: Start with deterministic redactions (email, phone, url). For names & addresses, use the Advanced workflow (server-side NLP).
        </div>
      </div>

      <div class="controls">
        <div style="margin-bottom:8px;font-weight:600">Redaction options</div>
        <label class="option"><input type="checkbox" id="chkEmail" checked> Emails</label>
        <label class="option"><input type="checkbox" id="chkPhone" checked> Phone numbers</label>
        <label class="option"><input type="checkbox" id="chkURL" checked> URLs</label>
        <label class="option"><input type="checkbox" id="chkIP" checked> IP addresses</label>
        <label class="option"><input type="checkbox" id="chkCC" checked> Credit-card-like numbers</label>
        <label class="option"><input type="checkbox" id="chkSSN"> US SSN / National IDs</label>
        <label class="option"><input type="checkbox" id="chkDOB"> Dates (DOBs: dd/mm/yyyy, yyyy-mm-dd)</label>
        <label class="option"><input type="checkbox" id="chkCustom"> Remove custom word(s)</label>
        <div id="customInputWrap" style="margin-top:6px;display:none">
          <input id="customWords" placeholder="Comma-separated words to remove" style="width:100%;padding:8px;border-radius:6px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:inherit">
        </div>

        <hr style="border:none;border-top:1px solid rgba(255,255,255,0.02);margin:10px 0">

        <div style="font-weight:600;margin-bottom:6px">Output</div>
        <textarea id="outputText" placeholder="Sanitized output will appear here..." readonly></textarea>

        <div class="outActions">
          <button id="copyBtn" class="btn-secondary">Copy</button>
          <button id="downloadBtn" class="btn-secondary">Download .txt</button>
          <button id="swapBtn" class="btn" style="background:linear-gradient(90deg,#f97316,#fb7185)">Overwrite Input</button>
        </div>

        <div class="footer">
          <div style="margin-top:10px"><strong>Note:</strong> This client-side tool uses pattern matching (regex) which is reliable for structured data (emails, phones, IPs). For intelligent redaction of names/addresses inside sentences, consider the Advanced server-side model.</div>
        </div>
      </div>
    </div>
  </div>

<script>
  // helpers
  const el = id => document.getElementById(id);
  const input = el('inputText');
  const output = el('outputText');
  const charCount = el('charCount');

  // patterns
  const patterns = {
    email: /([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})/g,
    url: /((https?:\/\/)?(www\.)?[-a-zA-Z0-9@:%._\+~#=]{2,256}\.[a-z]{2,6}\b([-a-zA-Z0-9@:%_\+.~#()?&//=;]*))/gi,
    ip: /\b(?:(?:2[0-4]\d|25[0-5]|1?\d?\d)(?:\.(?:2[0-4]\d|25[0-5]|1?\d?\d)){3})\b/g,
    // phone: flexible pattern to catch many intl & local formats (07xx, +91 98..., (123) 456-7890, 123-456-7890)
    phone: /(\+?\d{1,3}[-.\s]?)?(?:\(?\d{2,4}\)?[-.\s]?)?\d{3,4}[-.\s]?\d{3,4}/g,
    // credit card like: 13-19 digits usually grouped
    creditcard: /(?:\b(?:\d[ -]*?){13,19}\b)/g,
    ssn: /\b\d{3}-\d{2}-\d{4}\b/g,
    // dates: dd/mm/yyyy or yyyy-mm-dd or dd-mm-yyyy or mm/dd/yyyy
    date: /\b(?:\d{1,2}[\/\-]\d{1,2}[\/\-]\d{2,4}|\d{4}[\/\-]\d{1,2}[\/\-]\d{1,2})\b/g
  };

  // UI wiring
  el('charCount').innerText = 0;
  input.addEventListener('input', () => el('charCount').innerText = input.value.length);

  // custom words toggle
  el('chkCustom').addEventListener('change', e => {
    el('customInputWrap').style.display = e.target.checked ? 'block' : 'none';
  });

  function sanitizeText(text) {
    let out = text;

    // apply deterministic replacements based on checkboxes
    if (el('chkEmail').checked) out = out.replace(patterns.email, match => "[REDACTED_EMAIL]");
    if (el('chkURL').checked) out = out.replace(patterns.url, match => "[REDACTED_URL]");
    if (el('chkIP').checked) out = out.replace(patterns.ip, match => "[REDACTED_IP]");
    if (el('chkPhone').checked) {
      // phone regex is broad; avoid replacing short numeric sequences like '123' by checking length
      out = out.replace(patterns.phone, match => {
        // remove non-digits to check
        const digits = match.replace(/\D/g,'');
        if (digits.length >= 6) return "[REDACTED_PHONE]";
        return match; // keep short numbers
      });
    }
    if (el('chkCC').checked) {
      out = out.replace(patterns.creditcard, match => {
        // mask leaving last 4 digits if length plausible
        const digits = match.replace(/\D/g,'');
        if (digits.length >= 13 && digits.length <= 19) return "[REDACTED_CREDITCARD]";
        return match;
      });
    }
    if (el('chkSSN').checked) out = out.replace(patterns.ssn, "[REDACTED_SSN]");
    if (el('chkDOB').checked) out = out.replace(patterns.date, "[REDACTED_DATE]");

    // custom word removal
    if (el('chkCustom').checked) {
      const raw = el('customWords').value || '';
      const words = raw.split(',').map(s=>s.trim()).filter(Boolean);
      if (words.length > 0) {
        // build safe regex for each word (word boundaries)
        words.forEach(w => {
          try {
            const esc = w.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
            const re = new RegExp('\\b' + esc + '\\b', 'gi');
            out = out.replace(re, '[REDACTED_CUSTOM]');
          } catch(e) {
            console.warn('bad custom word', w);
          }
        });
      }
    }

    return out;
  }

  el('sanitizeBtn').addEventListener('click', () => {
    const txt = input.value || '';
    if (!txt.trim()) { alert('Paste some text first.'); return; }
    output.value = sanitizeText(txt);
    output.focus();
    output.select();
  });

  el('clearBtn').addEventListener('click', () => {
    if (confirm('Clear all input and output?')) { input.value=''; output.value=''; el('charCount').innerText=0; }
  });

  el('copyBtn').addEventListener('click', async () => {
    try {
      await navigator.clipboard.writeText(output.value || '');
      alert('Copied sanitized text to clipboard.');
    } catch (e) {
      alert('Copy failed. You can manually select and copy the text.');
    }
  });

  el('downloadBtn').addEventListener('click', () => {
    const data = output.value || '';
    const blob = new Blob([data], {type:'text/plain;charset=utf-8'});
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = 'sanitized.txt'; document.body.appendChild(a); a.click();
    a.remove(); URL.revokeObjectURL(url);
  });

  el('swapBtn').addEventListener('click', () => {
    if (!output.value) { alert('No sanitized output to copy back.'); return; }
    if (confirm('Overwrite input with sanitized output?')) {
      input.value = output.value;
      el('charCount').innerText = input.value.length;
    }
  });

  // initialization: prefill sample text (optional)
  input.value = "Sample: Contact me at john.doe@example.com or +91 98765-43210. Visit https://example.com. My DOB 12/05/1990. CC: 4111 1111 1111 1111. IP: 192.168.1.1";
  el('charCount').innerText = input.value.length;
</script>
</body>
</html>
