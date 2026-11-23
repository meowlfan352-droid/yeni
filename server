import http from 'node:http';
import { mkdirSync } from 'node:fs';
import { join, resolve } from 'node:path';
import { fileURLToPath } from 'node:url';
import { DatabaseSync } from 'node:sqlite';

// ======================================================
// SUNUCU VE VERİTABANI
// ======================================================

const root = fileURLToPath(new URL('.', import.meta.url));
const dataDir = resolve(process.env.DATA_DIR || join(root, '.local'));

mkdirSync(dataDir, { recursive: true });

const db = new DatabaseSync(
  join(dataDir, 'dengetabak-sohbet.sqlite')
);

db.exec(`
  CREATE TABLE IF NOT EXISTS feedback (
    id TEXT PRIMARY KEY,
    payload TEXT NOT NULL,
    created TEXT NOT NULL
  );

  CREATE TABLE IF NOT EXISTS comments (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    request_id TEXT NOT NULL UNIQUE,
    client TEXT NOT NULL,
    tray TEXT NOT NULL,
    day TEXT NOT NULL,
    portion INTEGER NOT NULL,
    adequacy TEXT NOT NULL,
    body TEXT NOT NULL,
    created TEXT NOT NULL,
    UNIQUE(client, tray, day)
  );
`);

function fail(message, status = 400) {
  const error = new Error(message);
  error.status = status;
  throw error;
}

function reply(res, status, data, type = 'application/json') {
  res.writeHead(status, {
    'Content-Type': type + '; charset=utf-8',
    'Cache-Control': 'no-store',
    'X-Content-Type-Options': 'nosniff'
  });

  res.end(
    type === 'application/json'
      ? JSON.stringify(data)
      : data
  );
}

function trayCode(value) {
  const code = String(value || '').trim().toUpperCase();

  if (!/^[A-Z0-9_-]{1,20}$/.test(code)) {
    fail('Tepsi kodunu kontrol et. Örnek: T025.');
  }

  return code;
}

function validKey(value) {
  return (
    typeof value === 'string' &&
    /^[A-Za-z0-9_-]{8,100}$/.test(value)
  );
}

function validatePortion(p) {
  if (![200, 250].includes(p.portion_g)) {
    fail('Küçük veya standart porsiyon seç.');
  }

  if (!['little', 'enough', 'much'].includes(p.adequacy)) {
    fail('Porsiyonun nasıl geldiğini seç.');
  }
}

async function readBody(req) {
  if (!req.headers['content-type']?.includes('application/json')) {
    fail('JSON verisi gerekli.');
  }

  if (req.headers.origin) {
    let origin;

    try {
      origin = new URL(req.headers.origin);
    } catch {
      fail('Geçersiz kaynak.', 403);
    }

    if (origin.host !== req.headers.host) {
      fail('Bu kaynağa izin verilmiyor.', 403);
    }
  }

  const chunks = [];
  let size = 0;

  for await (const chunk of req) {
    size += chunk.length;

    if (size > 8000) {
      fail('Veri çok uzun.', 413);
    }

    chunks.push(chunk);
  }

  let p;

  try {
    p = JSON.parse(Buffer.concat(chunks).toString('utf8'));
  } catch {
    fail('Veri okunamadı.');
  }

  if (!p || typeof p !== 'object' || Array.isArray(p)) {
    fail('Geçersiz veri.');
  }

  return p;
}

async function handle(req, res) {
  const url = new URL(req.url, 'http://' + req.headers.host);

  const pages = {
    '/ogrenci': 'student',
    '/mutfak': 'kitchen',
    '/yorumlar': 'comments'
  };

  if (pages[url.pathname]) {
    res.writeHead(302, {
      Location: '/' + url.search + '#' + pages[url.pathname]
    });

    return res.end();
  }

  if (
    req.method === 'GET' &&
    ['/', '/index.html'].includes(url.pathname)
  ) {
    return reply(res, 200, HTML, 'text/html');
  }

  // Ortak geri bildirimleri getir.
  if (
    req.method === 'GET' &&
    url.pathname === '/api/feedback'
  ) {
    const items = db.prepare(
      'SELECT payload FROM feedback ORDER BY created DESC LIMIT 1000'
    ).all().map(row => JSON.parse(row.payload));

    const total = db.prepare(
      'SELECT COUNT(*) AS n FROM feedback'
    ).get().n;

    return reply(res, 200, { items, total });
  }

  // Öğrenci geri bildirimini kaydet.
  if (
    req.method === 'POST' &&
    url.pathname === '/api/feedback'
  ) {
    const p = await readBody(req);

    validatePortion(p);

    if (!validKey(p.session)) {
      fail('Öğün kaydı geçersiz.');
    }

    const tray = trayCode(p.tray);
    const ratios = [0, 0.25, 0.5, 0.75, 1];

    if (
      !ratios.includes(p.leftover) ||
      !ratios.includes(p.extra_left)
    ) {
      fail('Kalan oranını kontrol et.');
    }

    if (!['before', 'after'].includes(p.phase)) {
      fail('Dönem geçersiz.');
    }

    const reasons = [
      'enough',
      'portion',
      'insufficient',
      'taste',
      'temperature',
      'time',
      'other'
    ];

    if (!reasons.includes(p.reason)) {
      fail('Neden seç.');
    }

    if (!['', 'yes', 'no'].includes(p.full)) {
      fail('Doygunluk yanıtı geçersiz.');
    }

    if (
      !Number.isFinite(p.extra_g) ||
      p.extra_g < 0 ||
      p.extra_g > 1000
    ) {
      fail('Ek servis miktarı geçersiz.');
    }

    const record = {
      session: p.session,
      tray,
      dish: 'pilav',
      phase: p.phase,
      portion_g: p.portion_g,
      adequacy: p.adequacy,
      leftover: p.leftover,
      reason: p.reason,
      full: p.full,
      extra_g: p.extra_g,
      extra_waste_g: p.extra_g * p.extra_left,
      submitted: new Date().toISOString(),
      data_status: 'shared_self_report'
    };

    const saved = db.prepare(
      'INSERT OR IGNORE INTO feedback VALUES (?, ?, ?)'
    ).run(
      record.session,
      JSON.stringify(record),
      record.submitted
    );

    return reply(res, 200, {
      ok: true,
      replayed: saved.changes === 0
    });
  }

  // Herkese açık yorumları getir.
  if (
    req.method === 'GET' &&
    url.pathname === '/api/comments'
  ) {
    const portion = url.searchParams.get('portion') || 'all';
    const cursor = url.searchParams.get('before') || '';

    if (
      !['all', '200', '250'].includes(portion) ||
      (cursor && !/^\d{1,15}$/.test(cursor))
    ) {
      fail('Geçersiz filtre.');
    }

    const conditions = [];
    const args = [];

    if (portion !== 'all') {
      conditions.push('portion = ?');
      args.push(Number(portion));
    }

    if (cursor) {
      conditions.push('id < ?');
      args.push(Number(cursor));
    }

    const rows = db.prepare(
      'SELECT id, portion AS portion_g, adequacy, body, ' +
      'created AS created_at FROM comments' +
      (conditions.length
        ? ' WHERE ' + conditions.join(' AND ')
        : '') +
      ' ORDER BY id DESC LIMIT 21'
    ).all(...args);

    const total = portion === 'all'
      ? db.prepare(
          'SELECT COUNT(*) AS n FROM comments'
        ).get().n
      : db.prepare(
          'SELECT COUNT(*) AS n FROM comments WHERE portion = ?'
        ).get(Number(portion)).n;

    const items = rows.slice(0, 20);

    return reply(res, 200, {
      items,
      total,
      next_cursor: rows.length > 20
        ? items.at(-1).id
        : null
    });
  }

  // Herkese açık yorum kaydet.
  if (
    req.method === 'POST' &&
    url.pathname === '/api/comments'
  ) {
    const p = await readBody(req);

    validatePortion(p);

    if (
      !validKey(p.client_key) ||
      !validKey(p.submission_key)
    ) {
      fail('Gönderim kaydı geçersiz.');
    }

    const tray = trayCode(p.tray);
    const text = typeof p.body === 'string'
      ? p.body.trim()
      : '';

    if (p.consent !== true) {
      fail('Herkese açık paylaşım onayı gerekli.');
    }

    if (
      Array.from(text).length < 3 ||
      Array.from(text).length > 400 ||
      /[\u0000-\u0008\u000b\u000c\u000e-\u001f]/.test(text)
    ) {
      fail('Yorum 3–400 karakter arasında olmalı.');
    }

    const old = db.prepare(
      'SELECT id FROM comments WHERE request_id = ?'
    ).get(p.submission_key);

    if (old) {
      return reply(res, 200, {
        ok: true,
        id: old.id
      });
    }

    const day = new Date().toLocaleDateString('en-CA', {
      timeZone: 'Europe/Istanbul'
    });

    const alreadyPosted = db.prepare(
      'SELECT id FROM comments ' +
      'WHERE client = ? AND tray = ? AND day = ?'
    ).get(p.client_key, tray, day);

    if (alreadyPosted) {
      fail(
        'Bu cihazdan bu tepsi için bugün yorum paylaşılmış.',
        409
      );
    }

    const recent = db.prepare(
      'SELECT COUNT(*) AS n FROM comments ' +
      'WHERE client = ? AND created >= ?'
    ).get(
      p.client_key,
      new Date(Date.now() - 60000).toISOString()
    );

    if (recent.n >= 3) {
      fail('Biraz bekleyip yeniden dene.', 429);
    }

    db.prepare(
      'INSERT INTO comments ' +
      '(request_id, client, tray, day, portion, adequacy, body, created) ' +
      'VALUES (?, ?, ?, ?, ?, ?, ?, ?)'
    ).run(
      p.submission_key,
      p.client_key,
      tray,
      day,
      p.portion_g,
      p.adequacy,
      text,
      new Date().toISOString()
    );

    return reply(res, 201, { ok: true });
  }

  reply(res, 404, { error: 'Adres bulunamadı.' });
}

// ======================================================
// ÖĞRENCİ PORTALI VE MUTFAK DEMOSU
// ======================================================

const HTML = String.raw`<!doctype html>
<html lang="tr">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>DengeTabak</title>

<style>
:root {
  --green: #176348;
  --ink: #173b31;
  --line: #dce3da;
  --pale: #e8f2e7;
  --muted: #65746d;
}

* { box-sizing: border-box; }
[hidden] { display: none !important; }

body {
  margin: 0;
  background: #f5f4ed;
  color: var(--ink);
  font: 16px/1.55 system-ui, sans-serif;
}

header {
  background: white;
  border-bottom: 1px solid var(--line);
  padding: 16px 5%;
  display: flex;
  align-items: center;
  gap: 14px;
}

.brand { font-size: 24px; font-weight: 800; }
.brand span { color: #d96b3c; }
.badge { margin-left: auto; font-size: 13px; color: var(--muted); }

main {
  max-width: 1150px;
  margin: auto;
  padding: 26px 20px;
}

h1 { font-size: 34px; line-height: 1.2; }
h2 { font-size: 24px; margin: 0 0 14px; }
h3 { margin: 8px 0 12px; }

.card {
  background: white;
  border: 1px solid var(--line);
  border-radius: 16px;
  padding: 24px;
  margin: 18px 0;
}

.grid {
  display: grid;
  grid-template-columns: 1.2fr 1fr;
  gap: 20px;
  align-items: start;
}

button, input, select, textarea { font: inherit; }
button, a { touch-action: manipulation; }

button {
  cursor: pointer;
  padding: 11px 16px;
  border: 1px solid var(--line);
  border-radius: 9px;
  background: white;
  color: var(--ink);
}

button:hover { border-color: var(--green); }
button:disabled { opacity: .55; cursor: default; }

.primary, .selected {
  background: var(--green);
  color: white;
}

label {
  display: block;
  font-size: 14px;
  font-weight: 650;
  margin: 14px 0 6px;
}

input, select, textarea {
  width: 100%;
  padding: 10px;
  border: 1px solid #bdcdbf;
  border-radius: 8px;
  color: var(--ink);
  background: white;
}

textarea { resize: vertical; }

input:focus,
select:focus,
textarea:focus,
button:focus-visible {
  outline: 3px solid #a3ceac;
  outline-offset: 2px;
}

.row {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  align-items: end;
}

.row > div { flex: 1; min-width: 150px; }

.choices {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.choices button { flex: 1; min-width: 65px; }
.muted { font-size: 14px; color: var(--muted); }

.notice {
  padding: 14px;
  background: #fff2df;
  border-left: 4px solid #d96b3c;
  border-radius: 6px;
}

.success {
  padding: 12px;
  background: var(--pale);
  border-radius: 8px;
}

.error { color: #a13124; }

.metrics {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}

.metric {
  padding: 18px;
  border: 1px solid var(--line);
  border-radius: 12px;
}

.metric strong { display: block; font-size: 28px; }

.tablewrap { overflow: auto; }
table { width: 100%; border-collapse: collapse; font-size: 14px; }

th, td {
  text-align: left;
  padding: 11px 8px;
  border-bottom: 1px solid var(--line);
}

.comment {
  border: 1px solid var(--line);
  padding: 18px;
  border-radius: 12px;
  margin: 14px 0;
  background: #fafcf9;
}

.comment p {
  white-space: pre-wrap;
  overflow-wrap: anywhere;
}

.tag {
  display: inline-block;
  background: var(--pale);
  padding: 5px 10px;
  border-radius: 20px;
  font-size: 14px;
}

.consent {
  display: flex;
  align-items: start;
  gap: 10px;
  font-weight: 400;
}

.consent input {
  width: 20px;
  flex: none;
  margin-top: 3px;
}

.links {
  display: flex;
  gap: 8px;
  margin: 18px 0;
  flex-wrap: wrap;
}

a { color: var(--green); }

.menu-toggle {
  width: 44px;
  height: 44px;
  padding: 0;
  font-size: 28px;
}

.student-menu {
  position: fixed;
  inset: 0 auto 0 0;
  margin: 0;
  width: min(320px, calc(100vw - 32px));
  height: 100dvh;
  max-height: 100dvh;
  border: 0;
  padding: 22px;
  color: var(--ink);
}

.student-menu::backdrop { background: #10241d66; }

.student-menu a {
  display: block;
  padding: 16px;
  border: 1px solid var(--line);
  border-radius: 10px;
  text-decoration: none;
  margin: 12px 0;
}

.student-menu a[aria-current] { background: var(--pale); }

.drawer-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.drawer-head h2 { font-size: 21px; }
.result { font-size: 34px; font-weight: 800; }

.calcgrid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0 14px;
}

details {
  border: 1px solid var(--line);
  border-radius: 9px;
  padding: 12px;
  margin: 14px 0;
}

summary { cursor: pointer; }
#print-report { display: none; }

@media (max-width: 760px) {
  .grid, .metrics, .calcgrid { grid-template-columns: 1fr; }
  h1 { font-size: 29px; }
  .card { padding: 18px; }
  .badge { font-size: 12px; }
  .brand { font-size: 22px; }
  .row > div { flex-basis: 100%; }
}

@media print {
  header, .hero, .panel, #kitchen-nav, dialog {
    display: none !important;
  }

  #print-report { display: block !important; }
  main { padding: 0; }
}
</style>
</head>

<body>

<header>
  <button
    id="menu-toggle"
    class="menu-toggle"
    aria-label="Menüyü aç"
    aria-controls="student-menu"
    aria-expanded="false"
  >⋮</button>

  <div class="brand">Denge<span>Tabak</span></div>
  <div class="badge" id="badge">Öğrenci portalı</div>
</header>

<dialog
  id="student-menu"
  class="student-menu"
  aria-labelledby="menu-title"
>
  <div class="drawer-head">
    <h2 id="menu-title">Öğrenci menüsü</h2>
    <button id="menu-close" aria-label="Menüyü kapat">✕</button>
  </div>

  <a href="#student" aria-label="Geri bildirim">
    Geri bildirim
  </a>

  <a href="#comments" aria-label="Öğrenci yorumları">
    Öğrenci yorumları
  </a>

  <p class="muted">Yorumları herkes okuyabilir.</p>
</dialog>

<main>

<div class="hero">
  <h1 id="title">Porsiyonun sana nasıl geldi?</h1>
  <p id="intro">
    Kısa bir geri bildirimle mutfağın daha uygun porsiyonlar
    sunmasına yardımcı ol.
  </p>
</div>

<nav id="kitchen-nav" class="links" hidden>
  <button data-tab="kitchen">Mutfak özeti</button>
  <button data-tab="impact">Etki hesabı</button>
</nav>

<!-- ÖĞRENCİ PORTALI -->

<section id="student" class="panel">
  <div class="grid">

    <form id="feedback-form" class="card">
      <span class="tag">Günün yemeği · Pilav</span>
      <h2>Porsiyonun nasıldı?</h2>

      <label for="student-tray">Tepsi kodun</label>
      <input
        id="student-tray"
        placeholder="Örn. T025"
        maxlength="20"
        required
      >

      <label for="portion">Aldığın porsiyon</label>
      <select id="portion">
        <option value="250">Standart · yaklaşık 250 g</option>
        <option value="200">Küçük · yaklaşık 200 g</option>
      </select>

      <label for="adequacy">Porsiyon sana nasıl geldi?</label>
      <select id="adequacy" required>
        <option value="">Seç</option>
        <option value="little">Az geldi</option>
        <option value="enough">Yeterliydi</option>
        <option value="much">Fazla geldi</option>
      </select>

      <label>İlk porsiyonun ne kadarı kaldı?</label>
      <div class="choices">
        <button type="button" data-left="0">Hiç</button>
        <button type="button" data-left="0.25">Çeyreği</button>
        <button type="button" data-left="0.5">Yarısı</button>
        <button type="button" data-left="0.75">Çoğu</button>
        <button type="button" data-left="1">Tamamı</button>
      </div>

      <label for="reason">Sence neden?</label>
      <select id="reason" required>
        <option value="">Seç</option>
        <option value="enough">Sorun yoktu</option>
        <option value="portion">Porsiyon fazlaydı</option>
        <option value="insufficient">Porsiyon az geldi</option>
        <option value="taste">Tadını beğenmedim</option>
        <option value="temperature">Yemek soğuktu</option>
        <option value="time">Vaktim yetmedi</option>
        <option value="other">Başka bir neden</option>
      </select>

      <label for="full">Doydun mu? (isteğe bağlı)</label>
      <select id="full">
        <option value="">Yanıtlamak istemiyorum</option>
        <option value="yes">Evet</option>
        <option value="no">Hayır</option>
      </select>

      <details>
        <summary>Ek servis aldıysan belirt</summary>

        <label for="extra-g">
          Ek servis miktarı (yaklaşık g)
        </label>
        <input
          id="extra-g"
          type="number"
          value="0"
          min="0"
          max="1000"
        >

        <label for="extra-left">Ek servisten kalan</label>
        <select id="extra-left">
          <option value="0">Hiç</option>
          <option value="0.25">Çeyreği</option>
          <option value="0.5">Yarısı</option>
          <option value="0.75">Çoğu</option>
          <option value="1">Tamamı</option>
        </select>
      </details>

      <p class="muted">
        Ad ve öğrenci numarası sorulmaz.
        Miktarlar reçete ve beyana dayalı tahmindir.
      </p>

      <button id="submit-feedback" class="primary">
        Geri bildirim gönder
      </button>

      <p id="feedback-message" role="status"></p>
    </form>

    <aside class="card">
      <h2>Yanıtın neyi değiştirecek?</h2>

      <p>
        Yanıtlar yemek ve porsiyona göre karşılaştırılır.
        Mutfak kontrollü bir porsiyon denemesi yapabilir;
        sonrasında sonuç yeniden incelenir.
      </p>

      <p class="muted">
        Porsiyon değişikliğine mutfak sorumlusu karar verir.
        Öğrenci yorumlarına sol üstteki üç noktadan ulaşabilirsin.
      </p>
    </aside>

  </div>
</section>

<!-- HERKESE AÇIK YORUMLAR -->

<section id="comments" class="panel" hidden>
  <div class="card">
    <h2>Öğrenci yorumları</h2>

    <p>
      Alacağın porsiyona karar verirken diğer öğrencilerin
      deneyimlerine bak.
    </p>

    <div class="links">
      <button data-comment-portion="all">Tümü</button>
      <button data-comment-portion="200">Küçük</button>
      <button data-comment-portion="250">Standart</button>
      <button id="refresh-comments">Yorumları yenile</button>
    </div>

    <p id="comments-count" class="muted"></p>
    <p id="comments-status" role="status"></p>

    <div id="comments-list"></div>
    <button id="more-comments" hidden>Daha fazla yorum</button>

    <form id="comment-form">
      <h2>Deneyimini paylaş</h2>

      <label for="comment-tray">Tepsi kodun</label>
      <input
        id="comment-tray"
        maxlength="20"
        placeholder="Örn. T025"
        required
      >

      <div class="row">
        <div>
          <label for="comment-portion">Aldığın porsiyon</label>

          <select id="comment-portion" required>
            <option value="">Seç</option>
            <option value="200">Küçük · 200 g</option>
            <option value="250">Standart · 250 g</option>
          </select>
        </div>

        <div>
          <label for="comment-adequacy">
            Porsiyon sana nasıl geldi?
          </label>

          <select id="comment-adequacy" required>
            <option value="">Seç</option>
            <option value="little">Az geldi</option>
            <option value="enough">Yeterliydi</option>
            <option value="much">Fazla geldi</option>
          </select>
        </div>
      </div>

      <label for="comment-body">Yorumun</label>
      <textarea
        id="comment-body"
        rows="4"
        minlength="3"
        maxlength="400"
        required
      ></textarea>

      <p class="muted">
        En fazla 400 karakter. Kişisel bilgi yazma.
        Tepsi kodun yorum kartında görünmez.
      </p>

      <label class="consent">
        <input id="comment-consent" type="checkbox" required>
        <span>
          Yorumumun herkese açık paylaşılmasını kabul ediyorum.
        </span>
      </label>

      <button id="submit-comment" class="primary">
        Yorumu paylaş
      </button>

      <p id="comment-message" role="status"></p>
    </form>
  </div>
</section>

<!-- MUTFAK DEMOSU -->

<section id="kitchen" class="panel" hidden>

  <p class="notice">
    Yarışma demosu: personel kimlik doğrulaması yoktur.
    Yanıtlar gönüllü beyanlardır; tartım veya kanıtlanmış
    tasarruf değildir.
  </p>

  <div class="card">
    <label for="analysis-basis">Gösterilecek yanıtlar</label>

    <select id="analysis-basis">
      <option value="manual">Tüm cihazlardan gelen yanıtlar</option>
      <option value="sample">Sentetik örnek veriler</option>
    </select>

    <p id="shared-status" class="muted"></p>
    <button id="refresh-feedback">Yanıtları yenile</button>
    <p id="data-counts" class="muted"></p>

    <div class="metrics">
      <div class="metric">
        Başlangıç
        <strong id="before-count">0</strong>
      </div>

      <div class="metric">
        Deneme
        <strong id="after-count">0</strong>
      </div>

      <div class="metric">
        Tahmini artık farkı
        <strong id="waste-difference">—</strong>
      </div>
    </div>
  </div>

  <div class="card">
    <h2>Porsiyon karşılaştırması</h2>

    <div class="tablewrap">
      <table>
        <thead>
          <tr>
            <th>Dönem</th>
            <th>Porsiyon</th>
            <th>Yanıt</th>
            <th>Tahmini artık/yanıt</th>
            <th>Aç kaldım</th>
          </tr>
        </thead>
        <tbody id="portion-summary"></tbody>
      </table>
    </div>

    <h3 id="rec-title"></h3>
    <p id="rec-body"></p>

    <p class="muted">
      20 yanıt eşiği demo kuralıdır;
      istatistiksel yeterlilik garantisi değildir.
    </p>

    <div class="row">
      <div>
        <label for="plan-meals">Denemedeki öğün sayısı</label>
        <input
          id="plan-meals"
          type="number"
          min="1"
          value="200"
        >
      </div>

      <div>
        <label for="trial-share">Küçük porsiyon payı (%)</label>
        <input
          id="trial-share"
          type="number"
          min="0"
          max="100"
          value="50"
        >
      </div>
    </div>

    <button id="approve" class="primary">
      Kontrollü denemeyi onayla
    </button>

    <p id="plan-result"></p>
    <div id="audit-list"></div>

    <details>
      <summary>Son gönderilen yanıtlar</summary>
      <div id="latest-feedback"></div>
    </details>
  </div>

  <div class="grid">
    <form id="production-form" class="card">
      <h2>Gün sonu kaydı</h2>

      <label for="production-portion">Porsiyon</label>
      <select id="production-portion">
        <option value="250">Standart · 250 g</option>
        <option value="200">Küçük · 200 g</option>
      </select>

      <label for="prepared">Hazırlanan porsiyon</label>
      <input id="prepared" type="number" min="0" value="200">

      <label for="served">Servis edilen porsiyon</label>
      <input id="served" type="number" min="0" value="190">

      <label for="discarded">
        Servis edilmeden atılan porsiyon
      </label>
      <input id="discarded" type="number" min="0" value="5">

      <button>Gün sonunu kaydet</button>
      <p id="production-message"></p>
    </form>

    <div class="card">
      <h2>Yeni tepsi kullanımı</h2>

      <label for="tray-id">Tepsi kodu</label>
      <input id="tray-id" value="T025" maxlength="20">

      <label for="session-phase">Dönem</label>
      <select id="session-phase">
        <option value="before">Başlangıç</option>
        <option value="after">Deneme</option>
      </select>

      <button id="new-session">
        Öğrenci bağlantısı oluştur
      </button>

      <p id="session-link"></p>

      <h3>Demo ve dışa aktarma</h3>

      <p class="muted">
        Örnekler 20 başlangıç ve 20 deneme kaydıdır;
        ortak yanıtlara karıştırılmaz.
      </p>

      <div class="links">
        <button id="load-demo">Örnek veriyi yükle</button>
        <button id="export-feedback">Yanıtları CSV indir</button>
        <button id="export-audit">İşlem kaydını indir</button>
      </div>

      <p class="muted">
        Plan, onay ve gün sonu kayıtları bu tarayıcıda saklanır.
      </p>
    </div>
  </div>
</section>

<!-- ETKİ HESABI -->

<section id="impact" class="panel" hidden>
  <p class="notice">
    Örnek senaryo: kampüs girdileri varsayımdır.
    Bu sayılar geri bildirimlerden otomatik alınmaz.
  </p>

  <div class="grid">
    <form id="impact-form" class="card">
      <h2>Etki hesabı girdileri</h2>
      <div id="calc-fields" class="calcgrid"></div>

      <label for="food_factor">
        Gıda katsayısı (kg CO₂e/kg), varsa
      </label>
      <input
        id="food_factor"
        type="number"
        min="0"
        step="any"
        placeholder="Boş bırakılabilir"
      >

      <label for="food_factor_source">
        Katsayının kaynağı ve kapsamı
      </label>
      <input id="food_factor_source">

      <button class="primary">Hesapla</button>
      <p id="calc-error" class="error"></p>
    </form>

    <div>
      <div class="card">
        <h2>Yıllık tahmini net gıda etkisi</h2>
        <div id="net-food" class="result"></div>
        <div id="calc-results"></div>
        <p id="carbon-note"></p>

        <p class="muted">
          Maliyet tasarrufu, üretim veya malzeme tüketimi
          gerçekten azaldığında doğrulanabilir.
        </p>
      </div>

      <div class="card">
        <button id="export-impact">Hesabı JSON indir</button>
        <button id="print-impact">Rapor yazdır</button>
      </div>
    </div>
  </div>
</section>

<section id="print-report"></section>
</main>

<script>
// ======================================================
// ORTAK ARAYÜZ İŞLEVLERİ
// ======================================================

const $ = id => document.getElementById(id);

const fmt = (n, digits = 1) =>
  Number(n).toLocaleString('tr-TR', {
    maximumFractionDigits: digits
  });

const esc = value =>
  String(value ?? '').replace(/[&<>"']/g, character => ({
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#39;'
  }[character]));

const uid = () =>
  globalThis.crypto?.randomUUID?.() ||
  ('id-' + Date.now() + '-' + Math.random().toString(36).slice(2));

function read(key, fallback) {
  try {
    return JSON.parse(localStorage.getItem(key)) ?? fallback;
  } catch {
    return fallback;
  }
}

const save = (key, value) =>
  localStorage.setItem(key, JSON.stringify(value));

const qs = new URLSearchParams(location.search);

const today = () =>
  new Date().toLocaleDateString('en-CA', {
    timeZone: 'Europe/Istanbul'
  });

const labels = {
  little: 'Az geldi',
  enough: 'Yeterliydi',
  much: 'Fazla geldi'
};

let rows = [];
let total = 0;
let samples = read('dt-samples', []);
let audit = read('dt-audit', []);
let production = read('dt-production', []);

let chosen = null;
let filter = 'all';
let cursor = null;
let commentRows = [];
let requestKey = uid();
let lastReport = null;

let client = read('dt-client', null);

if (!client) {
  client = uid();
  save('dt-client', client);
}

$('student-tray').value = qs.get('tray') || '';
$('comment-tray').value = $('student-tray').value;

async function api(path, body) {
  const response = await fetch(
    path,
    body
      ? {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(body)
        }
      : {}
  );

  const data = await response.json();

  if (!response.ok) {
    throw Error(data.error || 'İşlem başarısız.');
  }

  return data;
}

function message(id, text, ok = false) {
  $(id).textContent = text;
  $(id).className = ok ? 'success' : 'error';
}

// ======================================================
// ÜÇ NOKTA MENÜSÜ VE SAYFALAR
// ======================================================

const menu = $('student-menu');

$('menu-toggle').onclick = () => {
  menu.showModal();
  $('menu-toggle').setAttribute('aria-expanded', 'true');
};

$('menu-close').onclick = () => menu.close();

menu.onclose = () =>
  $('menu-toggle').setAttribute('aria-expanded', 'false');

menu.onclick = event => {
  const bounds = menu.getBoundingClientRect();

  if (event.target === menu && event.clientX > bounds.right) {
    menu.close();
  }
};

menu.querySelectorAll('a').forEach(link => {
  link.onclick = () => {
    menu.close();
    window.scrollTo(0, 0);
  };
});

function show() {
  let page = location.hash.slice(1) || 'student';

  if (!['student', 'comments', 'kitchen', 'impact'].includes(page)) {
    page = 'student';
  }

  document.querySelectorAll('.panel').forEach(panel => {
    panel.hidden = panel.id !== page;
  });

  const student = ['student', 'comments'].includes(page);

  $('menu-toggle').hidden = !student;
  $('kitchen-nav').hidden = student;

  $('badge').textContent = student
    ? 'Öğrenci portalı'
    : 'Mutfak demosu';

  $('title').textContent = {
    student: 'Porsiyonun sana nasıl geldi?',
    comments: 'Diğer öğrenciler ne diyor?',
    kitchen: 'Geri bildirimi mutfak kararına dönüştür.',
    impact: 'Hesabı adım adım incele.'
  }[page];

  $('intro').textContent = student
    ? 'Deneyimini paylaş, porsiyon tercihini diğer öğrencilerin yorumlarıyla destekle.'
    : 'Yanıtları, denemeleri ve örnek etki hesabını incele.';

  menu.querySelectorAll('a').forEach(link => {
    if (link.hash === '#' + page) {
      link.setAttribute('aria-current', 'page');
    } else {
      link.removeAttribute('aria-current');
    }
  });

  if (page === 'comments') loadComments();
  if (page === 'kitchen') refreshKitchen();
}

window.onhashchange = show;

document.querySelectorAll('[data-tab]').forEach(button => {
  button.onclick = () => {
    location.hash = button.dataset.tab;
  };
});

// ======================================================
// ÖĞRENCİ GERİ BİLDİRİMİ
// ======================================================

function meal() {
  const tray = $('student-tray').value.trim().toUpperCase();

  if (!/^[A-Z0-9_-]{1,20}$/.test(tray)) {
    throw Error('Tepsi kodunu kontrol et.');
  }

  $('student-tray').value = tray;

  const phase = qs.get('phase') === 'after'
    ? 'after'
    : 'before';

  const key =
    'dt-meal:' + today() + ':' +
    tray + ':' + phase + ':' +
    (qs.get('use') || 'default');

  let data = read(key, null);

  if (!data) {
    data = { id: uid(), sent: false };
    save(key, data);
  }

  return { tray, phase, key, data };
}

$('student-tray').onchange = () => {
  try {
    const current = meal();

    $('submit-feedback').disabled = current.data.sent;

    if (!$('comment-body').value) {
      $('comment-tray').value = current.tray;
    }
  } catch (error) {
    message('feedback-message', error.message);
  }
};

document.querySelectorAll('[data-left]').forEach(button => {
  button.onclick = () => {
    chosen = Number(button.dataset.left);

    document.querySelectorAll('[data-left]').forEach(other => {
      other.classList.toggle('selected', other === button);
    });
  };
});

$('feedback-form').onsubmit = async event => {
  event.preventDefault();
  $('submit-feedback').disabled = true;

  let sent = false;

  try {
    const current = meal();

    if (current.data.sent) {
      throw Error('Bu öğün için yanıt zaten gönderildi.');
    }

    if (chosen === null) {
      throw Error('Kalan miktarı seç.');
    }

    const record = {
      session: current.data.id,
      tray: current.tray,
      phase: current.phase,
      portion_g: Number($('portion').value),
      adequacy: $('adequacy').value,
      leftover: chosen,
      reason: $('reason').value,
      full: $('full').value,
      extra_g: Number($('extra-g').value),
      extra_left: Number($('extra-left').value)
    };

    await api('/api/feedback', record);

    current.data.sent = true;
    save(current.key, current.data);
    sent = true;

    message(
      'feedback-message',
      'Teşekkürler. Yanıtın ortak mutfak ekranına gönderildi.',
      true
    );

    if (!$('comment-body').value) {
      $('comment-tray').value = current.tray;
      $('comment-portion').value = String(record.portion_g);
      $('comment-adequacy').value = record.adequacy;
    }
  } catch (error) {
    message('feedback-message', error.message);
  } finally {
    $('submit-feedback').disabled = sent;
  }
};

try {
  if ($('student-tray').value) {
    $('submit-feedback').disabled = meal().data.sent;
  }
} catch {}

// ======================================================
// HERKESE AÇIK YORUMLAR
// ======================================================

async function loadComments(append = false) {
  try {
    const params = new URLSearchParams({ portion: filter });

    if (append && cursor) {
      params.set('before', cursor);
    }

    const data = await api('/api/comments?' + params);

    commentRows = append
      ? [...commentRows, ...data.items]
      : data.items;

    cursor = data.next_cursor;

    $('comments-count').textContent =
      data.total + ' yorum · ' +
      (
        filter === 'all'
          ? 'Tüm porsiyonlar'
          : filter === '200'
            ? 'Küçük porsiyon'
            : 'Standart porsiyon'
      );

    $('more-comments').hidden = !cursor;

    $('comments-list').innerHTML = commentRows.length
      ? commentRows.map(record =>
          '<article class="comment">' +
          '<span class="tag">Pilav · ' +
          (record.portion_g === 200 ? 'Küçük' : 'Standart') +
          ' · ' + record.portion_g + ' g</span>' +
          '<p class="muted">' +
          esc(labels[record.adequacy]) +
          '</p><p class="comment-body">' +
          esc(record.body) +
          '</p><small>Anonim öğrenci · ' +
          esc(new Date(record.created_at).toLocaleString('tr-TR')) +
          '</small></article>'
        ).join('')
      : '<p>Henüz bu porsiyon için yorum yok.</p>';

    $('comments-status').textContent =
      'Yorumlar öğrencilerin kendi beyanlarıdır.';
  } catch (error) {
    message('comments-status', error.message);
  }
}

document.querySelectorAll('[data-comment-portion]').forEach(button => {
  button.onclick = () => {
    filter = button.dataset.commentPortion;

    document.querySelectorAll('[data-comment-portion]').forEach(other => {
      other.classList.toggle('selected', other === button);
      other.setAttribute('aria-pressed', String(other === button));
    });

    cursor = null;
    loadComments();
  };
});

$('refresh-comments').onclick = () => loadComments();
$('more-comments').onclick = () => loadComments(true);

$('comment-form').oninput = () => {
  requestKey = uid();
};

$('comment-form').onsubmit = async event => {
  event.preventDefault();
  $('submit-comment').disabled = true;

  try {
    await api('/api/comments', {
      submission_key: requestKey,
      client_key: client,
      tray: $('comment-tray').value,
      portion_g: Number($('comment-portion').value),
      adequacy: $('comment-adequacy').value,
      body: $('comment-body').value,
      consent: $('comment-consent').checked
    });

    $('comment-body').value = '';
    $('comment-consent').checked = false;

    requestKey = uid();
    filter = 'all';
    cursor = null;

    message(
      'comment-message',
      'Yorumun paylaşıldı. Diğer öğrenciler de görebilir.',
      true
    );

    await loadComments();
  } catch (error) {
    message('comment-message', error.message);
  } finally {
    $('submit-comment').disabled = false;
  }
};

// ======================================================
// MUTFAK ÖZETİ VE ÖNERİLER
// ======================================================

const selected = () =>
  $('analysis-basis').value === 'sample'
    ? samples
    : rows;

function stats(list) {
  const count = list.length;

  const answered = list.filter(record =>
    ['yes', 'no'].includes(record.full)
  );

  return {
    n: count,

    mean: count
      ? list.reduce(
          (sum, record) =>
            sum +
            record.portion_g * record.leftover +
            (record.extra_waste_g || 0),
          0
        ) / count
      : 0,

    excess: count
      ? list.filter(record =>
          record.adequacy === 'much' ||
          record.reason === 'portion'
        ).length / count
      : 0,

    hunger: answered.length
      ? answered.filter(record => record.full === 'no').length /
        answered.length
      : null
  };
}

function renderKitchen() {
  const list = selected();

  const before = stats(
    list.filter(record => record.phase === 'before')
  );

  const after = stats(
    list.filter(record => record.phase === 'after')
  );

  $('before-count').textContent = before.n;
  $('after-count').textContent = after.n;

  $('waste-difference').textContent =
    before.n && after.n
      ? fmt(before.mean - after.mean) + ' g/yanıt'
      : '—';

  $('data-counts').textContent =
    'Ortak kayıtta ' + total + ' yanıt · ' +
    'Sentetik örnek ' + samples.length + ' yanıt' +
    (
      total > 1000
        ? ' · Analiz son 1000 ortak yanıtla sınırlı.'
        : ''
    );

  $('portion-summary').innerHTML =
    ['before', 'after'].flatMap(phase =>
      [250, 200].map(portion => {
        const summary = stats(
          list.filter(record =>
            record.phase === phase &&
            record.portion_g === portion
          )
        );

        return (
          '<tr><td>' +
          (phase === 'before' ? 'Başlangıç' : 'Deneme') +
          '</td><td>' + portion + ' g' +
          '</td><td>' + summary.n +
          '</td><td>' +
          (summary.n ? fmt(summary.mean) + ' g' : '—') +
          '</td><td>' +
          (
            summary.hunger === null
              ? '—'
              : fmt(summary.hunger * 100) + '%'
          ) +
          '</td></tr>'
        );
      })
    ).join('');

  let title = 'Yemek ve servis koşullarını incele';

  let text =
    'Porsiyonu değiştirmeden önce lezzet, sıcaklık ' +
    've servis koşullarını değerlendir.';

  if (before.n < 20) {
    title = 'Önce veri topla';
    text = 'Demo kuralı en az 20 başlangıç yanıtı bekler.';
  } else if (before.excess >= 0.5) {
    title = 'Küçük porsiyonu kontrollü dene';

    text =
      'Standart porsiyon ve ek servis seçeneği korunarak ' +
      'deneme önerilir.';

    if (after.n >= 20) {
      if (
        after.hunger !== null &&
        before.hunger !== null &&
        after.hunger > before.hunger
      ) {
        title = 'Doygunluğu yeniden değerlendir';

        text =
          'Aç kaldığını bildirenlerin oranı arttı. ' +
          'Yaygınlaştırmadan önce incele.';
      } else if (after.mean < before.mean) {
        title = 'Olumlu sinyal; kontrollü denemeyi sürdür';

        text =
          'Yanıt verenlerde tahmini artık azaldı; ' +
          'nedensellik ve kampüsün tamamı için etki ' +
          'kanıtlanmış değildir.';
      }
    }
  }

  $('rec-title').textContent = title;
  $('rec-body').textContent = text;
  $('approve').disabled = before.n < 20;

  $('latest-feedback').innerHTML =
    rows.slice(0, 5).map(record =>
      '<p>' + esc(record.tray) +
      ' · ' + record.portion_g + ' g · ' +
      esc(labels[record.adequacy]) + '</p>'
    ).join('') || 'Henüz ortak yanıt yok.';

  $('audit-list').innerHTML =
    audit.slice(-5).reverse().map(record =>
      '<p class="muted">' +
      esc(new Date(record.time).toLocaleString('tr-TR')) +
      ' · ' + esc(record.detail) + '</p>'
    ).join('');
}

async function refreshKitchen() {
  try {
    const data = await api('/api/feedback');

    rows = data.items;
    total = data.total;

    renderKitchen();

    $('shared-status').textContent =
      'Ortak kayıtlar güncellendi.';
  } catch (error) {
    message('shared-status', error.message);
  }
}

$('refresh-feedback').onclick = refreshKitchen;
$('analysis-basis').onchange = renderKitchen;

// ======================================================
// SENTETİK DEMO VERİLERİ
// ======================================================

$('load-demo').onclick = () => {
  samples = [];

  for (const phase of ['before', 'after']) {
    for (let i = 0; i < 20; i++) {
      samples.push({
        session: 'sample-' + phase + '-' + i,
        tray: 'DEMO',
        phase,

        portion_g: phase === 'before' ? 250 : 200,

        leftover: phase === 'before'
          ? (i < 16 ? 0.25 : 0.5)
          : (i < 16 ? 0.25 : 0),

        adequacy:
          phase === 'before' && i < 16
            ? 'much'
            : 'enough',

        reason:
          phase === 'before' && i < 16
            ? 'portion'
            : 'enough',

        full:
          i < (phase === 'before' ? 18 : 19)
            ? 'yes'
            : 'no',

        extra_waste_g: 0,
        data_status: 'synthetic_demo'
      });
    }
  }

  save('dt-samples', samples);
  $('analysis-basis').value = 'sample';

  renderKitchen();
};

// ======================================================
// MUTFAK PLANI VE GÜN SONU
// ======================================================

$('approve').onclick = () => {
  const count = Number($('plan-meals').value);
  const share = Number($('trial-share').value);

  if (
    !Number.isInteger(count) ||
    count < 1 ||
    share < 0 ||
    share > 100 ||
    !Number.isFinite(share)
  ) {
    return message('plan-result', 'Girdileri kontrol et.');
  }

  const small = Math.round(count * share / 100);
  const standard = count - small;

  const planned =
    (small * 200 + standard * 250) / 1000;

  const detail =
    small + ' küçük + ' + standard + ' standart: ' +
    fmt(planned) + ' kg ilk servis. ' +
    'Tamamı standart olsa ' + fmt(count * 0.25) + ' kg. ' +
    'Fark ' + fmt(count * 0.25 - planned) +
    ' kg; bu gerçekleşmiş tasarruf değildir. ' +
    'Ek servis ayrıca planlanır.';

  audit.push({
    time: new Date().toISOString(),
    detail,
    basis: $('analysis-basis').value
  });

  save('dt-audit', audit);

  renderKitchen();
  $('plan-result').textContent = detail;
};

$('production-form').onsubmit = event => {
  event.preventDefault();

  const prepared = Number($('prepared').value);
  const served = Number($('served').value);
  const discarded = Number($('discarded').value);
  const grams = Number($('production-portion').value);

  if (
    ![prepared, served, discarded].every(
      number => Number.isInteger(number) && number >= 0
    ) ||
    served + discarded > prepared
  ) {
    return message(
      'production-message',
      'Servis edilen + atılan miktar, hazırlananı aşamaz.'
    );
  }

  production.push({
    time: new Date().toISOString(),
    portion_g: grams,
    prepared,
    served,
    discarded,
    estimated_discarded_kg: discarded * grams / 1000
  });

  save('dt-production', production);

  message(
    'production-message',
    'Kaydedildi. Atılan yaklaşık ' +
    fmt(discarded * grams / 1000) +
    ' kg. Kalan ' +
    (prepared - served - discarded) +
    ' porsiyon otomatik olarak atık sayılmadı.',
    true
  );
};

$('new-session').onclick = () => {
  const tray = $('tray-id').value.trim().toUpperCase();

  if (!/^[A-Z0-9_-]{1,20}$/.test(tray)) {
    return message('session-link', 'Tepsi kodunu kontrol et.');
  }

  const url = new URL(location.href);

  url.search = new URLSearchParams({
    tray,
    phase: $('session-phase').value,
    use: uid()
  }).toString();

  url.hash = 'student';

  $('session-link').innerHTML =
    '<a target="_blank" rel="noopener" href="' +
    esc(url.href) +
    '">Yeni öğrenci kaydını aç</a>';
};

// ======================================================
// DIŞA AKTARMA
// ======================================================

function download(name, data, type = 'application/json') {
  const blob = new Blob(
    [data],
    { type: type + ';charset=utf-8' }
  );

  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');

  link.href = url;
  link.download = name;
  link.click();

  setTimeout(() => URL.revokeObjectURL(url), 1000);
}

$('export-feedback').onclick = () => {
  const columns = [
    'session',
    'tray',
    'phase',
    'portion_g',
    'adequacy',
    'leftover',
    'reason',
    'full',
    'extra_waste_g',
    'data_status'
  ];

  const quote = value =>
    '"' + String(value ?? '').replace(/"/g, '""') + '"';

  const csv = '\ufeff' + [
    columns.map(quote).join(','),
    ...selected().map(record =>
      columns.map(key => quote(record[key])).join(',')
    )
  ].join('\r\n');

  download('yanitlar.csv', csv, 'text/csv');
};

$('export-audit').onclick = () =>
  download(
    'islem-kaydi.json',
    JSON.stringify({ audit, production }, null, 2)
  );

// ======================================================
// ETKİ HESABI
// ======================================================

const inputs = [
  ['before_g', 'Önceki artık (g/öğün)', 75],
  ['after_g', 'Sonraki artık (g/öğün)', 40],
  ['rebound_g', 'Henüz içerilmeyen ek atık (g/öğün)', 5],
  ['users', 'Kapsamdaki öğün sayısı / servis', 200],
  ['days', 'Bu yemeğin yıllık servis sayısı', 180],
  ['system_days', 'Sistemin yıllık çalışma günü', 180],
  ['cost_per_kg', 'Varsayılan malzeme maliyeti (TL/kg)', 60],
  ['operating_cost_year', 'Yıllık ek sistem gideri (TL)', 1800],
  ['energy_kwh_day', 'Ek elektrik (kWh/gün)', 0.1],
  ['electricity_factor', 'Elektrik katsayısı (kg CO₂e/kWh)', 0.469],
  ['other_carbon_year', 'Diğer sistem yükü (kg CO₂e/yıl)', 2]
];

$('calc-fields').innerHTML = inputs.map(
  ([key, label, value]) =>
    '<div><label for="' + key + '">' + label + '</label>' +
    '<input id="' + key +
    '" type="number" min="0" step="any" required value="' +
    value + '"></div>'
).join('');

function calculate() {
  try {
    const p = {};

    for (const [key] of inputs) {
      if ($(key).value === '') {
        throw Error('Girdiler boş bırakılamaz.');
      }

      p[key] = Number($(key).value);

      if (!Number.isFinite(p[key]) || p[key] < 0) {
        throw Error('Girdiler sıfır veya pozitif olmalı.');
      }
    }

    if (p.days > 366 || p.system_days > 366) {
      throw Error('Gün sayısı 366’yı aşamaz.');
    }

    const meals = p.users * p.days;

    const gross =
      (p.before_g - p.after_g) * meals / 1000;

    const rebound =
      p.rebound_g * meals / 1000;

    const net = gross - rebound;

    const own =
      p.energy_kwh_day *
      p.system_days *
      p.electricity_factor +
      p.other_carbon_year;

    const factorText = $('food_factor').value;
    const source = $('food_factor_source').value.trim();

    let carbon = null;

    if (factorText !== '') {
      const factor = Number(factorText);

      if (
        !source ||
        !Number.isFinite(factor) ||
        factor < 0
      ) {
        throw Error(
          'Geçerli gıda katsayısını ve kaynağını gir.'
        );
      }

      carbon = net * factor - own;
    }

    const results = {
      annual_meals: meals,
      gross_food_kg: gross,
      extra_waste_kg: rebound,
      net_food_kg: net,
      potential_gross_cost: net * p.cost_per_kg,
      potential_net_cost:
        net * p.cost_per_kg - p.operating_cost_year,
      own_carbon_kg: own,
      net_carbon_kg: carbon
    };

    lastReport = {
      status: 'example_scenario_not_measured',
      formula:
        '(before_g-after_g-rebound_g)/1000*users*days',
      inputs: p,
      food_factor: factorText,
      food_factor_source: source,
      results
    };

    $('net-food').textContent = fmt(net) + ' kg/yıl';

    $('calc-results').innerHTML =
      '<p>Brüt azalma: ' + fmt(gross) + ' kg</p>' +
      '<p>Ek atık payı: ' + fmt(rebound) + ' kg</p>' +
      '<p>Potansiyel brüt malzeme değeri: ' +
      fmt(results.potential_gross_cost) + ' TL</p>' +
      '<p>Ek gider sonrası potansiyel sonuç: ' +
      fmt(results.potential_net_cost) + ' TL</p>' +
      '<p>Sistemin tahmini yükü: ' +
      fmt(own, 3) + ' kg CO₂e/yıl</p>';

    $('carbon-note').textContent = carbon === null
      ? (
          'Tarife uygun gıda katsayısı eksik; ' +
          'net karbon kazancı doğrulanmadı. Denklem: ' +
          fmt(net) + ' × EF_gıda − ' +
          fmt(own, 3) + ' kg CO₂e.'
        )
      : (
          'Kaynağı girilmiş katsayıyla net karbon senaryosu: ' +
          fmt(carbon, 3) +
          ' kg CO₂e/yıl. Kaynağın uygunluğu ayrıca doğrulanmalı.'
        );

    $('calc-error').textContent = '';

    return true;
  } catch (error) {
    lastReport = null;

    $('net-food').textContent = 'Hesap geçersiz';
    $('calc-results').textContent = '';
    $('carbon-note').textContent = '';
    $('calc-error').textContent = error.message;

    return false;
  }
}

$('impact-form').onsubmit = event => {
  event.preventDefault();
  calculate();
};

$('export-impact').onclick = () => {
  if (calculate()) {
    download(
      'etki-hesabi.json',
      JSON.stringify(lastReport, null, 2)
    );
  }
};

$('print-impact').onclick = () => {
  if (!calculate()) return;

  const report = lastReport;

  $('print-report').innerHTML =
    '<h1>DengeTabak · Etki senaryosu</h1>' +
    '<p>ÖRNEK VARSAYIMLAR — GERÇEK PİLOT SONUCU DEĞİLDİR.</p>' +
    '<p>Yıllık etki = (önceki artık − sonraki artık − ek atık payı)' +
    ' / 1000 × öğün × yıllık servis.</p><table>' +
    inputs.map(([key, label]) =>
      '<tr><td>' + esc(label) + '</td><td>' +
      esc(report.inputs[key]) + '</td></tr>'
    ).join('') +
    '</table><h2>' +
    fmt(report.results.net_food_kg) +
    ' kg/yıl</h2><p>' +
    esc($('carbon-note').textContent) +
    '</p><p>Gönüllü beyanlar tartım değildir. ' +
    'Maliyet kazancı üretim veya malzeme azalmasıyla doğrulanmalıdır. ' +
    'Sonraki artık ek servisi zaten içeriyorsa ek atık tekrar düşülmez.</p>';

  window.print();
};

// Açık mutfak ekranını otomatik yenile.
setInterval(() => {
  if (document.hidden) return;

  if (location.hash === '#kitchen') {
    refreshKitchen();
  }
}, 5000);

// Yorumların ilk sayfasını otomatik yenile.
setInterval(() => {
  if (
    !document.hidden &&
    location.hash === '#comments' &&
    commentRows.length <= 20
  ) {
    loadComments();
  }
}, 10000);

calculate();
show();
</script>

</body>
</html>`;

// ======================================================
// SUNUCUYU BAŞLAT
// ======================================================

const port = Number(process.env.PORT || 8787);

const server = http.createServer((req, res) => {
  handle(req, res).catch(error => {
    if (!error.status) {
      console.error(error.message);
    }

    reply(res, error.status || 500, {
      error: error.status
        ? error.message
        : 'Sunucu hatası. Lütfen yeniden dene.'
    });
  });
});

server.on('error', error => {
  console.error(
    error.code === 'EADDRINUSE'
      ? 'Port kullanımda. Önce çalışan sunucuyu durdur.'
      : error.message
  );
});

server.listen(
  port,
  process.env.HOST || '127.0.0.1',
  () => console.log('Local: http://127.0.0.1:' + port)
);
