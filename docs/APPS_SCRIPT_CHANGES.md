# Perubahan Apps Script untuk portal Gracie

Panduan ini untuk pemilik portal, bukan untuk programmer. Ikuti urutannya. Jangan lompat ke Deploy sebelum langkah cadangan dan Script Properties selesai.

Website baru **tidak lagi** membawa kata sandi HR atau kunci ImgBB. HR dan staff masuk dengan nomor WhatsApp dan PIN seperti biasa. Setelah PIN benar, Apps Script membuat **tiket sesi** yang berlaku 6 jam. Browser menyimpan tiket itu hanya di tab yang sedang terbuka, lalu mengirimnya lewat POST (bukan di alamat URL).

Kata sandi HR yang dulu tertulis di website **sudah bocor**. Kata sandi itu tidak diketik di website lagi. Ia hanya disimpan di Script Properties, dan hanya dipakai Apps Script sebagai stempel untuk tiket sesi. HR tetap masuk dengan PIN akunnya. Kalau kolom role di sheet adalah `hr`, tiketnya boleh membuka menu HR. Staff biasa hanya bisa membuka datanya sendiri.

## Sebelum mulai

Siapkan waktu sekali duduk. Urutannya:

1. Cadangkan kode Apps Script.
2. Isi Script Properties dengan kata sandi **baru** dan garam PIN.
3. Ubah kode Apps Script seperti di bawah.
4. Deploy **versi baru** pada deployment yang sudah dipakai website (alamat URL tidak berubah).
5. Baru setelah itu tayangkan website baru (gabungkan pull request / deploy Vercel).
6. Tes login staff dan HR.
7. Arsipkan deployment lama.
8. Cabut kunci ImgBB.
9. Naikkan `SESSION_EPOCH` kalau ada tiket percobaan yang ingin diputus.
10. Repo boleh dibuat private **setelah** Vercel jalan. Private tidak menggantikan langkah 7 dan 8.

Kalau langkah 4 sudah di-deploy tetapi website yang tayang masih yang lama, portal akan gagal login sampai website baru tayang. Kalau website baru tayang lebih dulu, yang muncul: "Server belum mengeluarkan sesi."

Untuk mengembalikan keadaan lama: di Apps Script buka Deploy → Manage deployments → pensil pada deployment aktif → Version → pilih versi sebelum hari ini → Deploy. Lalu jangan tayangkan website baru.

Jangan menulis kata sandi baru, garam PIN, atau kunci ImgBB di file kode, di GitHub, di email, atau di chat. Dokumen ini sengaja tidak memuat nilainya.

## 1. Cadangkan

1. Buka spreadsheet portal → **Extensions** → **Apps Script**.
2. Pilih semua kode (`Ctrl+A` / `Cmd+A`), salin, tempel ke dokumen Google yang hanya Anda yang bisa membuka. Judulnya misalnya "Cadangan Apps Script portal".
3. Di Apps Script: **Deploy** → **Manage deployments**. Catat deployment mana yang URL-nya berisi ID aktif di bawah. Itu yang **jangan** diarsipkan.

ID deployment yang dipakai `index.html` sekarang (yang ini yang dipertahankan, hanya versinya yang diperbarui):

`AKfycbyDX_SwOXv8AlbWK17l_JQzbAZfD_K3OxhnhATOErLde0lyKaxm0ZdR5fRDCDbzr2yx`

## 2. Script Properties

1. Di Apps Script: ikon gerigi **Project Settings**.
2. Bagian **Script properties** → **Add script property**. Tambahkan tiga baris. Nama properti harus persis seperti ini (huruf besar, garis bawah):

| Nama properti | Isi |
|---|---|
| `HR_PASSWORD` | Kata sandi baru yang panjang, minimal 16 karakter, **bukan** kata sandi yang dulu ada di website. Ini stempel server. Tidak ada orang yang mengetiknya di portal. |
| `PIN_SALT` | Kalimat acak lain, juga minimal 16 karakter, **berbeda** dari `HR_PASSWORD`. Dipakai untuk menyimpan PIN sebagai hash. Setelah staff login sekali, jangan diganti, kalau tidak semua PIN hash rusak. |
| `SESSION_EPOCH` | Angka `1`. Nanti, untuk memaksa semua orang login ulang, ubah menjadi `2`, lalu `3`, dan seterusnya. Tidak perlu deploy ulang. |

3. Simpan. Jangan salin isinya ke tempat lain.

## 3. Ganti nama fungsi lama

Di editor kode, cari `function doGet`.

1. Ubah **hanya namanya** menjadi `function routeGetLama`. Isi di dalamnya jangan dihapus dulu.
2. Kalau ada `function doPost`, ubah namanya menjadi `function routePostLama`. Portal lama hampir pasti tidak memakainya. Jangan dihapus; biarkan menganggur.
3. Simpan. Kalau muncul error karena `doGet` tidak ada, itu normal sampai langkah 4 ditempel.

## 4. Tempel blok pengaman

Tempel seluruh blok di bawah ini di **bagian paling atas** file, di atas `routeGetLama`. Lalu simpan.

Kalau Apps Script menolak dengan pesan fungsi sudah ada (`doGet` declared twice), berarti langkah 3 terlewat.

Dua baris yang mungkin perlu Anda ubah ada di paling atas blok: `SHEET_CUTI` dan `SHEET_REIMBURSE`. Isi dengan nama tab yang persis terlihat di bawah spreadsheet. Huruf besar dan spasi harus sama. Kalau ragu, biarkan dulu, tes di langkah 10, lalu perbaiki kalau pesannya menyebut nama tab.

```javascript
// ==== PENGAMAN PORTAL GRACIE (tempel utuh) ====
// Nama tab. Ubah kalau tidak sama dengan spreadsheet Anda.
var SHEET_CUTI = 'Cuti';
var SHEET_REIMBURSE = 'Reimburse';

var SESI_MS = 6 * 60 * 60 * 1000;
var TIKET_PIN_MS = 15 * 60 * 1000;

var AKSI_HR = {
  getAllStaff: true, addStaff: true, updateStaffHR: true,
  approveCuti: true, rejectCuti: true, undoCuti: true,
  approveReimburse: true, rejectReimburse: true, undoReimburse: true,
  addKategori: true, updateKategori: true, addJenisCuti: true, updateJenisCuti: true,
  resetPin: true, getGajiChart: true, getGaji: true, getTemplateGaji: true,
  getKomponenGaji: true, updateGaji: true, addGaji: true, syncGajiFromSheets: true,
  generateGaji: true, setTemplateGaji: true, addKomponenGaji: true, updateKomponenGaji: true
};

var AKSI_STAFF = {
  getStaff: true, getCuti: true, getReimburse: true, getCutiSummary: true,
  addCuti: true, updateCuti: true, deleteCuti: true,
  addReimburse: true, updateReimburse: true, deleteReimburse: true,
  uploadReceipt: true, updateStaff: true, changePin: true,
  getGajiChartStaff: true, getGajiStaff: true
};

var FIELD_PROFIL_STAFF = {
  NamaLengkap: true, TanggalLahir: true, NIK: true, WA: true, Email: true,
  Alamat: true, NoRekening: true, NamaBank: true, TelegramID: true
};

var AKSI_CEK_BARIS = {
  updateCuti: { sheet: 'cuti', pending: true },
  deleteCuti: { sheet: 'cuti', pending: true },
  updateReimburse: { sheet: 'reimburse', pending: true },
  deleteReimburse: { sheet: 'reimburse', pending: true }
};

function doGet(e) {
  var params = (e && e.parameter) ? e.parameter : {};
  return cekAksesPortal(params, 'GET');
}

function doPost(e) {
  var raw = (e && e.postData && e.postData.contents) ? e.postData.contents : '';
  var params;
  try {
    params = raw ? JSON.parse(raw) : {};
  } catch (err) {
    return balasJson_({ ok: false, error: 'Body tidak valid' });
  }
  if (!params || typeof params !== 'object') {
    return balasJson_({ ok: false, error: 'Body tidak valid' });
  }
  return cekAksesPortal(params, 'POST');
}

function cekAksesPortal(params, method) {
  try {
    var hasil = putuskanAksi_(params || {}, method);
    if (hasil && hasil.lanjut) {
      return routeGetLama({ parameter: jadiString_(params) });
    }
    return balasJson_(hasil || { ok: false, error: 'Aksi tidak dikenal' });
  } catch (err) {
    try { console.error(err && err.stack ? err.stack : err); } catch (abaikan) {}
    return balasJson_({ ok: false, error: 'Gangguan server. Cek Script Properties HR_PASSWORD dan PIN_SALT.' });
  }
}

function putuskanAksi_(params, method) {
  var action = String(params.action || '');

  if (action === 'logout') {
    if (method !== 'POST') return { ok: false, error: 'Gunakan POST' };
    var sesKeluar = bacaSesi(params.token);
    if (sesKeluar) cabutSesi_(params.token, sesKeluar.exp);
    return { ok: true };
  }

  if (action === 'sessionInfo') {
    if (method !== 'POST') return { ok: false, error: 'Gunakan POST' };
    var siapa = bacaSesi(params.token);
    if (!siapa) return { ok: false, auth: false, error: 'Sesi habis. Silakan masuk lagi.' };
    return { ok: true, staffId: siapa.staffId, role: siapa.role };
  }

  if (action === 'loginStaff') {
    if (params.pin && method !== 'POST') return { ok: false, error: 'Gunakan POST' };
    return { lanjut: true };
  }

  if (action === 'setPin') {
    if (method !== 'POST') return { ok: false, error: 'Gunakan POST' };
    var tiket = bacaTiketPin(params.setupToken);
    if (!tiket || tiket.staffId !== String(params.staffId || '')) {
      return { ok: false, error: 'Sesi pembuatan PIN tidak berlaku. Masukkan nomor WA lagi.' };
    }
    return { lanjut: true };
  }

  if (action === 'getKategori' || action === 'getJenisCuti') {
    var all = String(params.all || '').toLowerCase();
    if (all === 'true' || all === '1') return wajibHr_(params, method);
    return { lanjut: true };
  }

  if (action === 'getLiburRentang') return { lanjut: true };

  if (AKSI_HR[action]) return wajibHr_(params, method);
  if (AKSI_STAFF[action]) return wajibStaff_(params, method, action);

  return { ok: false, error: 'Aksi tidak dikenal' };
}

function wajibHr_(params, method) {
  if (method !== 'POST') return { ok: false, error: 'Gunakan POST' };
  var ses = bacaSesi(params.token);
  if (!ses) return { ok: false, auth: false, error: 'Sesi habis. Silakan masuk lagi.' };
  if (ses.role !== 'hr') return { ok: false, error: 'Tidak punya akses HR' };
  return { lanjut: true };
}

function wajibStaff_(params, method, action) {
  if (method !== 'POST') return { ok: false, error: 'Gunakan POST' };
  var ses = bacaSesi(params.token);
  if (!ses) return { ok: false, auth: false, error: 'Sesi habis. Silakan masuk lagi.' };
  if (ses.role !== 'hr') {
    if (!params.staffId || String(params.staffId) !== String(ses.staffId)) {
      return { ok: false, error: 'Sesi tidak sesuai' };
    }
    if (action === 'updateStaff') {
      for (var k in params) {
        if (!params.hasOwnProperty(k)) continue;
        if (k === 'action' || k === 'token' || k === 'staffId') continue;
        if (!FIELD_PROFIL_STAFF[k]) return { ok: false, error: 'Field tidak boleh diubah' };
      }
    }
    var cekBaris = pastikanBarisMilik_(action, params, ses);
    if (cekBaris) return cekBaris;
  }
  return { lanjut: true };
}

function pastikanBarisMilik_(action, params, ses) {
  var aturan = AKSI_CEK_BARIS[action];
  if (!aturan) return null;
  if (!params.id) return { ok: false, error: 'Data tidak lengkap' };
  var namaTab = aturan.sheet === 'cuti' ? SHEET_CUTI : SHEET_REIMBURSE;
  var baris = bacaBarisSheet_(namaTab, params.id);
  if (baris.error) return { ok: false, error: baris.error };
  if (String(baris.StaffID) !== String(ses.staffId)) {
    return { ok: false, error: 'Bukan data Anda' };
  }
  if (aturan.pending && String(baris.Status || '').toLowerCase() !== 'pending') {
    return { ok: false, error: 'Hanya pengajuan Pending yang bisa diubah' };
  }
  return null;
}

function bacaBarisSheet_(namaTab, id) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  if (!ss) {
    return { error: 'Script ini harus terikat ke spreadsheet portal (Extensions → Apps Script dari file Sheet).' };
  }
  var sh = ss.getSheetByName(namaTab);
  if (!sh) {
    var nama = ss.getSheets().map(function (s) { return s.getName(); }).join(', ');
    return { error: 'Tab "' + namaTab + '" tidak ketemu. Tab yang ada: ' + nama + '. Perbaiki SHEET_CUTI atau SHEET_REIMBURSE lalu deploy ulang.' };
  }
  var values = sh.getDataRange().getValues();
  if (!values.length) return { error: 'Tab kosong' };
  var header = values[0];
  var kolomId = indeksHeader_(header, ['id']);
  var kolomStaff = indeksHeader_(header, ['staffid', 'staff id', 'id staff']);
  var kolomStatus = indeksHeader_(header, ['status']);
  if (kolomId < 0 || kolomStaff < 0 || kolomStatus < 0) {
    return { error: 'Kolom ID, StaffID, atau Status tidak ada di tab ' + namaTab };
  }
  var ingin = String(id);
  for (var r = 1; r < values.length; r++) {
    if (String(values[r][kolomId]) === ingin) {
      return {
        StaffID: values[r][kolomStaff],
        Status: values[r][kolomStatus]
      };
    }
  }
  return { error: 'Data tidak ditemukan' };
}

function indeksHeader_(header, nama) {
  for (var i = 0; i < header.length; i++) {
    var h = String(header[i] || '').trim().toLowerCase();
    for (var n = 0; n < nama.length; n++) {
      if (h === nama[n]) return i;
    }
  }
  return -1;
}

function jadiString_(params) {
  var out = {};
  for (var k in params) {
    if (!params.hasOwnProperty(k)) continue;
    if (params[k] === undefined || params[k] === null) continue;
    out[k] = String(params[k]);
  }
  return out;
}

function balasJson_(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}

function rahasiaStempel_() {
  var secret = PropertiesService.getScriptProperties().getProperty('HR_PASSWORD') || '';
  if (!secret || secret.length < 16) {
    throw new Error('NO_SECRET');
  }
  return secret;
}

function buatSesiStaff(staffId, role) {
  return buatTiket_({
    purpose: 'session',
    staffId: String(staffId),
    role: String(role || 'staff'),
    exp: Date.now() + SESI_MS
  });
}

function buatTiketPin(staffId) {
  return buatTiket_({
    purpose: 'setup',
    staffId: String(staffId),
    role: 'setup',
    exp: Date.now() + TIKET_PIN_MS
  });
}

function buatTiket_(body) {
  var secret = rahasiaStempel_();
  body.v = 1;
  body.epoch = PropertiesService.getScriptProperties().getProperty('SESSION_EPOCH') || '1';
  body.nonce = Utilities.getUuid();
  var payload = b64Encode_(JSON.stringify(body));
  return payload + '.' + tanda_(payload, secret);
}

function bacaSesi(token) {
  var body = bacaTiket_(token);
  if (!body || body.purpose !== 'session') return null;
  if (cache_().get('rev_' + body._sig)) return null;
  return { staffId: String(body.staffId), role: String(body.role), exp: Number(body.exp) };
}

function bacaTiketPin(token) {
  var body = bacaTiket_(token);
  if (!body || body.purpose !== 'setup') return null;
  return { staffId: String(body.staffId), exp: Number(body.exp) };
}

function bacaTiket_(token) {
  var secret = '';
  try { secret = rahasiaStempel_(); } catch (e) { return null; }
  var tokenStr = String(token || '');
  if (!tokenStr || tokenStr.indexOf('.') < 0) return null;
  if (samaRata_(tokenStr, secret)) return null;
  var dot = tokenStr.indexOf('.');
  var payload = tokenStr.slice(0, dot);
  var sig = tokenStr.slice(dot + 1);
  if (!samaRata_(sig, tanda_(payload, secret))) return null;
  var body;
  try { body = JSON.parse(b64Decode_(payload)); } catch (err) { return null; }
  if (!body || !body.exp || Date.now() > Number(body.exp)) return null;
  var epoch = PropertiesService.getScriptProperties().getProperty('SESSION_EPOCH') || '1';
  if (String(body.epoch) !== String(epoch)) return null;
  body._sig = sig;
  return body;
}

function cabutSesi_(token, exp) {
  var dot = String(token).indexOf('.');
  if (dot < 0) return;
  var sig = String(token).slice(dot + 1);
  var ttl = Math.floor((Number(exp) - Date.now()) / 1000);
  if (ttl < 1) return;
  if (ttl > 21600) ttl = 21600;
  cache_().put('rev_' + sig, '1', ttl);
}

function cache_() {
  return CacheService.getScriptCache();
}

function kunciPinSalah(wa) {
  return Number(cache_().get('pintry_' + String(wa)) || '0') >= 5;
}

function catatPinSalah(wa) {
  var key = 'pintry_' + String(wa);
  var n = Number(cache_().get(key) || '0') + 1;
  cache_().put(key, String(n), 900);
}

function resetCatatanPin(wa) {
  cache_().remove('pintry_' + String(wa));
}

function hashPinStaff(pin, staffId) {
  var salt = PropertiesService.getScriptProperties().getProperty('PIN_SALT') || '';
  if (!salt || salt.length < 16) throw new Error('NO_SALT');
  var digest = Utilities.computeDigest(
    Utilities.DigestAlgorithm.SHA_256,
    salt + '|' + String(staffId) + '|' + String(pin),
    Utilities.Charset.UTF_8
  );
  return 'sha256:' + Utilities.base64Encode(digest);
}

function cocokkanPin(pinTersimpan, pinDiketik, staffId) {
  var s = String(pinTersimpan || '');
  if (!s) return false;
  if (s.indexOf('sha256:') === 0) return samaRata_(s, hashPinStaff(pinDiketik, staffId));
  return samaRata_(s, String(pinDiketik));
}

function pinMasihTeksBiasa(pinTersimpan) {
  var s = String(pinTersimpan || '');
  return s !== '' && s.indexOf('sha256:') !== 0;
}

function tanda_(payload, secret) {
  var bytes = Utilities.computeHmacSha256Signature(String(payload), String(secret));
  return Utilities.base64EncodeWebSafe(bytes).replace(/=+$/, '');
}

function b64Encode_(text) {
  return Utilities.base64EncodeWebSafe(String(text), Utilities.Charset.UTF_8).replace(/=+$/, '');
}

function b64Decode_(text) {
  var bytes = Utilities.base64DecodeWebSafe(padB64_(text));
  return Utilities.newBlob(bytes).getDataAsString('UTF-8');
}

function padB64_(s) {
  var str = String(s);
  var mod = str.length % 4;
  if (mod === 0) return str;
  return str + new Array(5 - mod).join('=');
}

function samaRata_(a, b) {
  var aa = String(a);
  var bb = String(b);
  if (aa.length !== bb.length) return false;
  var ha = Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, aa, Utilities.Charset.UTF_8);
  var hb = Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, bb, Utilities.Charset.UTF_8);
  var diff = 0;
  for (var i = 0; i < ha.length; i++) diff |= (ha[i] ^ hb[i]);
  return diff === 0;
}

function tesRahasiaPortal() {
  var p = PropertiesService.getScriptProperties();
  var hr = p.getProperty('HR_PASSWORD') || '';
  var salt = p.getProperty('PIN_SALT') || '';
  Logger.log('HR_PASSWORD terisi dan cukup panjang: ' + (hr.length >= 16));
  Logger.log('PIN_SALT terisi dan cukup panjang: ' + (salt.length >= 16));
  Logger.log('SESSION_EPOCH: ' + (p.getProperty('SESSION_EPOCH') || '1'));
  if (hr.length < 16 || salt.length < 16) {
    Logger.log('BERHENTI: isi Script Properties dulu.');
    return;
  }
  var token = buatSesiStaff('TES', 'hr');
  var ses = bacaSesi(token);
  Logger.log('sesi tes berhasil dibaca: ' + (ses && ses.role === 'hr' && ses.staffId === 'TES'));
  Logger.log('kata sandi mentah ditolak sebagai token: ' + (bacaSesi(hr) === null));
  var setup = buatTiketPin('TES');
  var tiket = bacaTiketPin(setup);
  Logger.log('tiket PIN terbaca: ' + (tiket && tiket.staffId === 'TES'));
  Logger.log('tiket PIN tidak lolos sebagai sesi HR: ' + (bacaSesi(setup) === null));
  Logger.log('Jangan menyalin isi log ini ke mana pun. Tidak ada kata sandi di log kalau langkah ini diikuti.');
}
```

Di Project Settings, pastikan **Enable Chrome V8 runtime** aktif, lalu jalankan `tesRahasiaPortal` dari menu fungsi di editor (pilih nama itu, lalu **Run**). Pertama kali, Google minta izin. Buka **Execution log**. Semua baris harus `true`, dan dua baris panjang harus `true`. Kalau `BERHENTI`, kembali ke langkah 2.

Log ini tidak menampilkan kata sandi. Jangan menambah `Logger.log` yang mencetak `HR_PASSWORD`, PIN, atau tiket.

## 5. Hapus kata sandi dari dalam kode lama

Masih di `routeGetLama` (kode lama Anda):

1. Cari tulisan `HR_PASSWORD` atau baris yang menyimpan kata sandi lama (yang dulu sama dengan yang tertulis di website). **Hapus baris itu.** Jangan pindahkan nilainya ke file ini.
2. Cari setiap perbandingan token dengan kata sandi itu. Bentuknya kira-kira `if (token != ...)` atau `if (e.parameter.token != ...)`. **Hapus seluruh if itu**, termasuk `return` di dalamnya. Pemeriksaan baru sudah dilakukan `cekAksesPortal` sebelum kode lama berjalan. Kalau if itu dibiarkan, tiket sesi yang sah akan ditolak dan menu HR kosong.
3. Cari `imgbb` atau `IMGBB`. Kalau ada di Apps Script, hapus juga. Struk tidak lagi lewat ImgBB.
4. Jangan membandingkan `e.parameter.token` dengan Script Properties. Token dari browser adalah tiket, bukan kata sandi.
5. Di Project Settings, **Enable Chrome V8 runtime** harus aktif (itu bawaan proyek baru).

Perhatikan fungsi yang dipakai bersama staff dan HR: `getCuti` dan `getReimburse`. Dulu "ada token" artinya HR, jadi semua baris dikembalikan. Sekarang staff juga membawa token.

- Kalau `e.parameter.staffId` terisi, kembalikan hanya baris dengan StaffID itu.
- Kalau `staffId` kosong, permintaan itu dari HR (blok baru sudah menolak staff yang tidak mengirim `staffId`). Boleh kembalikan semua.

`getGaji`, `getAllStaff`, dan aksi lain di daftar HR tidak dipanggil staff. Menghapus perbandingan kata sandi di situ aman, karena blok baru sudah menolak tiket yang bukan `hr`.

`deleteCuti` / `deleteReimburse` staff dicek blok baru: baris harus milik mereka dan statusnya Pending. HR tetap boleh menghapus lewat tiket `hr`.

Kode lama membaca `e.parameter` seperti waktu masih GET. Blok baru menyalin isi POST ke `e.parameter`, jadi `uploadReceipt` yang membaca `e.parameter.base64` biasanya tidak perlu diubah. Foto besar sekarang muat, karena tidak lagi menumpang di URL.

## 6. Ubah login, buat PIN, dan ganti PIN

Role harus dibaca dari sheet, bukan dari data yang dikirim browser. Nilai untuk HR harus tetap `hr` (huruf kecil), sama seperti yang portal lama sudah terima.

### Saat nomor WA dicek dan PIN masih kosong

Di jawaban yang sekarang berisi `needSetPin: true`, tambahkan `setupToken`. Contoh field-nya:

```javascript
needSetPin: true,
staffId: staffId,
nama: nama,
role: role,
setupToken: buatTiketPin(staffId)
```

Kirim `setupToken` **hanya** kalau sel PIN staff itu benar-benar kosong. Jangan kirim pada jawaban PIN yang sudah ada, dan jangan kirim `sessionToken` sebelum PIN benar.

### Saat PIN diketik

Di awal pengecekan PIN (nomor `wa` yang sudah dinormalisasi, bentuk `628...`):

```javascript
if (kunciPinSalah(wa)) {
  // pakai cara return JSON yang sudah ada di fungsi Anda
  // { ok: false, error: 'Terlalu banyak percobaan. Coba lagi dalam 15 menit.' }
}
```

Ganti perbandingan PIN polos (`!=` atau `==`) dengan:

```javascript
if (!cocokkanPin(pinTersimpan, pinYangDiketik, staffId)) {
  catatPinSalah(wa);
  // return { ok: false, error: 'PIN salah' }
}
resetCatatanPin(wa);
if (pinMasihTeksBiasa(pinTersimpan)) {
  // timpa sel PIN dengan hashPinStaff(pinYangDiketik, staffId)
}
```

`pinTersimpan` adalah isi sel PIN di sheet. `pinYangDiketik` adalah PIN dari permintaan. `staffId` adalah ID baris staff itu.

Pada jawaban sukses, tambahkan tiket:

```javascript
ok: true,
staffId: staffId,
nama: nama,
role: role,
sessionToken: buatSesiStaff(staffId, role)
```

Tanpa `sessionToken`, website menampilkan "Server belum mengeluarkan sesi."

### Saat staff membuat PIN pertama (`setPin`)

Blok baru sudah menolak permintaan tanpa `setupToken` yang sah. Di dalam fungsi lama, tambahkan dua hal **sebelum** menulis PIN:

1. Kalau sel PIN sudah terisi, tolak. Jangan menimpa. Balas `{ ok: false, error: 'PIN sudah ada. Masuk dengan PIN, atau minta HR reset.' }`.
2. Simpan `hashPinStaff(pinBaru, staffId)`, bukan angka PIN-nya.

Jawaban sukses wajib membawa `sessionToken: buatSesiStaff(staffId, role)` dan `role`.

`resetPin` (dari HR) mengosongkan sel PIN. Jangan menulis hash ke situ. Staff yang di-reset akan mendapat `setupToken` pada login berikutnya karena selnya kosong.

### Saat staff mengganti PIN (`changePin`)

1. `cocokkanPin` untuk PIN lama. Kalau salah, `catatPinSalah`. Jangan mengganti PIN.
2. Kalau benar, simpan `hashPinStaff(pinBaru, staffId)` dan `resetCatatanPin`.
3. Abaikan `staffId` yang tidak sama dengan tiket. Blok baru sudah menolaknya; fungsi lama jangan menimpa PIN orang lain.

## 7. Deploy versi baru tanpa mengubah URL

1. **Deploy** → **Manage deployments**.
2. Pilih deployment yang URL-nya berisi ID aktif di langkah 1. Jangan memilih yang lain.
3. Pensil → **Version** → **New version**. Keterangan: `sesi HR, tanpa sandi di website`.
4. **Execute as:** Me. **Who has access:** Anyone.
5. **Deploy**. URL harus tetap sama. Jangan membuat New deployment untuk yang dipakai website, karena URL-nya akan berubah dan `index.html` tidak mengenalinya.

"Anyone" di sini artinya browser pengunjung boleh memanggil alamat itu tanpa login Google. Data tidak terbuka karena tiket sesi. Jangan mengganti menjadi Only myself: portal akan rusak total.

Aksi HR, aksi yang mengubah data, login dengan PIN, unggah struk, dan data pribadi staff hanya diterima lewat POST. GET dengan `token`, `pin`, atau `base64` ditolak.

Website mengirim POST dengan header `Content-Type: text/plain` dan isi JSON. Jangan mengganti header itu menjadi `application/json`: browser akan memblokir permintaan (CORS preflight) dan portal terlihat "koneksi gagal". `doPost` di blok baru sudah membaca JSON dari `e.postData.contents`. Skrip yang sekarang sudah menjawab POST bentuk ini.

## 8. Tayangkan website, lalu tes

Setelah deploy Apps Script, gabungkan pull request website (atau deploy Vercel dari branch yang sama) di waktu yang sama.

Tes di jendela penyamaran:

1. Staff biasa masuk dengan WA dan PIN. Beranda, cuti, reimburse, dan gaji sendiri tampil.
2. Staff mengunggah struk. Harus tersimpan tanpa ImgBB. Kalau gagal, yang tampil pesan gagal, bukan pengajuan tanpa struk.
3. Staff membatalkan cuti atau reimburse yang masih Pending miliknya. Harus berhasil.
4. Akun HR masuk dengan WA dan PIN-nya (bukan kata sandi Script Properties). Dashboard terisi, setujui atau tolak satu pengajuan percobaan, buka menu gaji.
5. Tutup tab, buka portal lagi. Harus diminta login lagi.
6. Di tab yang sama, refresh. Harus tetap masuk.
7. Di alamat portal, coba buka `?action=getAllStaff&token=apapun`. Tidak boleh ada daftar staff. Yang benar: penolakan JSON, atau bukan data.

Kalau staff membatalkan cuti lalu pesannya menyebut nama tab, ubah `SHEET_CUTI` atau `SHEET_REIMBURSE` sesuai nama yang disebut pesan itu, lalu deploy ulang versi baru pada deployment yang sama.

Kalau menu HR kosong dan pesannya soal akses atau sesi, kembali ke langkah 5: masih ada perbandingan kata sandi lama.

## 9. Arsipkan deployment lama

Riwayat git menyimpan beberapa URL Apps Script lama. Selama deployment itu tidak diarsipkan, siapa pun yang punya URL itu masih bisa memanggil kode lama, termasuk yang memercayai kata sandi yang sudah bocor.

**Deploy** → **Manage deployments**. Arsipkan setiap deployment yang URL-nya **bukan** ID aktif. Termasuk deployment uji coba kalau Anda sempat membuat New deployment.

ID yang harus diarsipkan (dari riwayat git):

- `AKfycbyeT_G6yv78bVMWS0pOzP4phsUFgVn3-tLrvBvRxXsKbRmxUFXLT3JiSCfsomMngPfeCw`
- `AKfycby3l5nIPgwQRpGEmdZmeLTiT_gxZYdZQvdDPcnuExuehionsTiOEBTEFc3XsxuT9r36`
- `AKfycbwH0UoI-DOypRSCt1f_w2bNMtk3VH86d6z-UISTMZIfi70xeunjqNhINPq8QZrJ4gXo`
- `AKfycbzOW592L-7R56mQh3CUN7JYIbjOGjguemHwm15txzw0zMHyc0rh8x5GRIA2WdBd46lz`
- `AKfycbzg_nUi1mIMeq-hadEQEsbvKEK3AQune16NnWNfjuAEZe8a2B3pLkblK6VN1LO0ndm4`
- `AKfycbwNU_cBKUUH1fVOXo1KGq1mXM6UCQ7ckDba8-SYrLfTN2Z5e5HMS5essV-kLTlj9Ram`

Kalau ada deployment lain yang tidak ada di daftar ini dan bukan yang aktif, arsipkan juga.

Setelah itu buka satu URL lama di jendela penyamaran. Yang benar: pesan Google bahwa aplikasi tidak ada atau tidak berhak. Bukan data JSON. Kalau masih JSON, deployment itu belum diarsipkan.

Jangan mengarsipkan deployment dengan ID `AKfycbyDX_SwOXv8AlbWK17l_JQzbAZfD_K3OxhnhATOErLde0lyKaxm0ZdR5fRDCDbzr2yx`.

## 10. Cabut kunci ImgBB dan putuskan sesi lama

1. Masuk ke akun ImgBB yang membuat kunci API.
2. Di halaman API ImgBB, hapus atau ganti kunci itu. Website tidak memanggil ImgBB lagi. Struk masuk ke Google Drive lewat `uploadReceipt`.
3. Kalau ada orang lain yang pernah membuka kode website, naikkan `SESSION_EPOCH` di Script Properties (dari `1` menjadi `2`). Semua tiket langsung mati. Tidak perlu deploy ulang.

Kata sandi lama tidak perlu "dihapus dari Google" selain dengan tidak memakainya lagi. Yang baru hanya ada di Script Properties. Kode lama yang masih membandingkan kata sandi bocor menjadi tidak berguna setelah deployment lama diarsipkan.

## 11. Vercel dan repo private

Website adalah file statis di akar repo. `index.html` adalah halaman utama. `dev.html` hanya mengalihkan ke `index.html` dan tidak berisi aplikasi lama.

Di Vercel, saat import:

- Framework Preset: **Other**
- Build Command: kosong
- Output Directory: kosong (akar repo)
- Install Command: kosong

Tidak ada `vercel.json`. Tidak perlu. Domain `gracieportal.xyz` diarahkan setelah tes di URL Vercel berhasil.

Membuat repo menjadi private **tidak** menghapus salinan yang sudah terunduh, dan tidak mematikan deployment Apps Script lama. Lakukan langkah 9 dan 10 dulu, baru ubah visibilitas repo. Jangan mengubah visibilitas dari pull request ini.

## Kelemahan login staff

Yang ditutup oleh blok baru, asal langkah 6 ikut dipasang. Kondisi skrip yang sekarang, tanpa mengetes akun sungguhan: `getAllStaff` tanpa tiket ditolak (`Unauthorized`), tetapi `setPin` tanpa tiket tetap dijalankan (ID yang tidak ada dibalas "Staff tidak ditemukan", bukan ditolak karena belum login), dan `getGajiStaff` tanpa tiket dibalas `ok`. Karena itu pemeriksaan baru tidak boleh dilewati.

- `setPin` hanya dengan `staffId`. Dulu siapa pun yang tahu ID staff bisa memasang PIN. Sekarang butuh tiket 15 menit, dan tiket itu hanya dibuat kalau sel PIN kosong. Tiket yang diulang setelah PIN terisi ditolak.
- Menghapus atau mengubah cuti/reimburse orang lain dengan menebak ID. Staff hanya bisa yang statusnya Pending dan `StaffID`-nya sama dengan tiket.
- Membaca gaji, cuti, atau profil orang lain hanya dengan `staffId`. Permintaan tanpa tiket, atau tiket yang tidak cocok, ditolak.
- Staff mengubah role atau PIN lewat simpan profil. Field profil yang boleh diubah dibatasi (nama, tanggal lahir, NIK, WA, email, alamat, rekening, bank, Telegram).
- PIN dan tiket di alamat URL. Login dengan PIN, ganti PIN, unggah struk, dan semua aksi HR sekarang POST.
- Staff membatalkan pengajuan dengan kata sandi HR. Sekarang memakai tiket staff.

Yang masih terbuka, dan tidak bisa ditutup hanya dari file website:

- PIN 4 digit tetap pendek. Batas 5 salah per 15 menit per nomor mengurangi tebakan, tetapi enam digit lebih aman. Minta staff memakai 6 digit saat ganti PIN.
- Orang yang sudah tahu nomor WA bisa mengetahui nomor itu terdaftar dan melihat nama sebelum PIN. Sapaan "Halo, nama" butuh itu. Tidak ada data gaji pada langkah itu.
- Tiket berlaku 6 jam. Logout menghapus tiket di cache Google selama sisa masa itu. Kalau cache Google terhapus sendiri, tiket masih sah sampai habis waktunya. Menaikkan `SESSION_EPOCH` mematikan semuanya sekaligus.
- Siapa pun yang bisa membuka Google Sheet tetap bisa membaca data, termasuk PIN yang belum di-hash. Setelah tiap orang login sekali, sel PIN berubah menjadi `sha256:...`. Jangan mengirim file sheet ke luar.
- Alamat Apps Script tetap bisa dipanggil siapa saja. Yang tidak mereka punya adalah stempel di Script Properties, jadi mereka tidak bisa membuat tiket sendiri.

## Daftar aksi yang website kirim

Bebas, tanpa tiket, GET boleh: `getKategori` dan `getJenisCuti` tanpa `all`, serta `getLiburRentang`. `loginStaff` tanpa PIN juga lewat sini, tetapi PIN wajib POST.

POST dengan tiket staff (HR boleh untuk staff lain): `getStaff`, `getCuti`, `getReimburse`, `getCutiSummary`, `addCuti`, `updateCuti`, `deleteCuti`, `addReimburse`, `updateReimburse`, `deleteReimburse`, `uploadReceipt`, `updateStaff`, `changePin`, `getGajiChartStaff`, `getGajiStaff`.

POST dengan tiket role `hr` saja: `getAllStaff`, `addStaff`, `updateStaffHR`, `approveCuti`, `rejectCuti`, `undoCuti`, `approveReimburse`, `rejectReimburse`, `undoReimburse`, `addKategori`, `updateKategori`, `addJenisCuti`, `updateJenisCuti`, `resetPin`, `getGaji`, `getGajiChart`, `getTemplateGaji`, `setTemplateGaji`, `getKomponenGaji`, `addKomponenGaji`, `updateKomponenGaji`, `addGaji`, `updateGaji`, `generateGaji`, `syncGajiFromSheets`, serta `getKategori` / `getJenisCuti` dengan `all=true`.

`sessionInfo` dan `logout` ditangani blok baru, tidak masuk ke kode lama.

Kalau suatu hari ada aksi baru, tambahkan namanya ke `AKSI_HR` atau `AKSI_STAFF`. Aksi yang tidak terdaftar ditolak.
