# Panduan Pembuatan Modul CRUD Mini — UjiBase Mini

**Proyek:** UjiBase Mini  
**Misi:** CRUD Sederhana dengan Pure PHP + MySQLi (Tanpa Framework)

Panduan ini menuntun pembuatan proyek CRUD mini dari nol: mulai dari database, koneksi, hingga modul CRUD berjalan penuh di browser.

---

## Langkah 0: Persiapan Folder Proyek

Buat folder proyek baru di dalam `htdocs` (XAMPP) atau `www` (Laragon):

```
ujibase_mini/
├── index.php              ← Dashboard utama (halaman pertama yang dibuka)
├── koneksi.php
├── partials/
│   └── header.php         ← Sidebar + layout wrapper (di-include semua halaman)
├── obat/
│   ├── index.php
│   ├── store.php
│   ├── edit.php
│   ├── update.php
│   └── delete.php
└── resep/
    ├── index.php
    ├── store.php
    ├── edit.php
    ├── update.php
    └── delete.php
```

---

## Langkah 1: Setup Database di phpMyAdmin

Buka **phpMyAdmin**, masuk ke tab **SQL** (di halaman utama phpMyAdmin, **bukan** di dalam database lain), lalu jalankan kueri berikut satu kali:

```sql
-- 1. Buat database baru
CREATE DATABASE IF NOT EXISTS db_ujibase_mini;
USE db_ujibase_mini;

-- 2. Buat tabel obat super simpel
CREATE TABLE obat (
    id INT AUTO_INCREMENT PRIMARY KEY,
    kode_obat VARCHAR(20) NOT NULL,
    nama_obat VARCHAR(100) NOT NULL,
    harga_jual INT NOT NULL,
    stok INT NOT NULL
);

-- 3. Masukkan 1 data dummy agar tabel tidak kosong
INSERT INTO obat (kode_obat, nama_obat, harga_jual, stok)
VALUES ('OBT-001', 'Paracetamol 500mg', 5000, 150);

-- 4. Tambah beberapa data obat sungguhan untuk modal awal
INSERT INTO obat (kode_obat, nama_obat, harga_jual, stok)
VALUES
('OBT-002', 'Amoxicillin 500mg', 7500, 100),
('OBT-003', 'Ibuprofen 400mg', 6000, 80),
('OBT-004', 'Lansoprazole 30mg', 12000, 50);

-- 5. Buat tabel resep versi Super Simple (Tanpa tabel master pasien & dokter)
CREATE TABLE resep (
    id INT AUTO_INCREMENT PRIMARY KEY,
    tanggal_resep DATE NOT NULL,
    nama_pasien VARCHAR(100) NOT NULL,
    nama_dokter VARCHAR(100) NOT NULL,
    obat_id INT NOT NULL,
    jumlah INT NOT NULL,
    keterangan TEXT,
    CONSTRAINT fk_resep_obat FOREIGN KEY (obat_id) REFERENCES obat(id) ON DELETE RESTRICT ON UPDATE CASCADE
);

-- 6. Masukkan 1 data dummy resep untuk tes UI nanti
INSERT INTO resep (tanggal_resep, nama_pasien, nama_dokter, obat_id, jumlah, keterangan)
VALUES ('2026-05-30', 'Ahmad Syaifuddin', 'dr. Budi Santoso', 1, 2, 'Diminum 3x sehari sesudah makan');
```

> **Catatan:** Tabel `resep` sengaja menggunakan kolom teks biasa untuk `nama_pasien` dan `nama_dokter` — tidak ada tabel master pasien/dokter — agar proyek ini benar-benar _super simpel_.

---

## Langkah 2: Membuat File Koneksi

Buat file `koneksi.php` langsung di dalam folder `ujibase_mini/`.

```php
<?php
// koneksi.php
$host = "localhost";
$user = "root";
$pass = "";
$db   = "db_ujibase_mini";
$koneksi = mysqli_connect($host, $user, $pass, $db);

// Cek jika koneksi gagal
if (!$koneksi) {
    die("SYSTEM ERROR // KONEKSI DATABASE GAGAL: " . mysqli_connect_error());
}
?>
```

> **Akses proyek:** `http://localhost/ujibase_mini/`

---

## Langkah 3: Membuat Partial Header (Sidebar Navigasi)

Buat folder `partials/` di dalam `ujibase_mini/`, lalu buat file `header.php` di dalamnya.  
File ini berisi layout sidebar yang di-`include` oleh **semua** halaman — obat, resep, dan edit sekalipun.

```php
<?php
// partials/header.php
// Deteksi halaman aktif berdasarkan URL path
$current = $_SERVER['PHP_SELF'];
$is_obat  = strpos($current, '/obat/')  !== false;
$is_resep = strpos($current, '/resep/') !== false;
$is_home  = !$is_obat && !$is_resep;
?>
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>UjiBase Mini</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body { font-family: sans-serif; display: flex; min-height: 100vh; background: #f3f4f6; }

        /* Sidebar */
        .sidebar {
            width: 220px;
            background: #134e4a;
            color: #ccfbf1;
            display: flex;
            flex-direction: column;
            padding: 0;
            position: fixed;
            top: 0; left: 0;
            height: 100vh;
        }
        .sidebar .brand {
            padding: 20px 18px 16px;
            font-size: 16px;
            font-weight: bold;
            color: #fff;
            border-bottom: 1px solid #0f3d39;
            letter-spacing: 0.5px;
        }
        .sidebar .brand span { font-size: 11px; font-weight: normal; color: #99f6e4; display: block; margin-top: 2px; }
        .sidebar nav { padding: 12px 0; flex: 1; }
        .sidebar nav a {
            display: block;
            padding: 10px 18px;
            font-size: 14px;
            color: #a7f3d0;
            text-decoration: none;
            transition: background 0.15s;
            border-left: 3px solid transparent;
        }
        .sidebar nav a:hover { background: #0f3d39; color: #fff; }
        .sidebar nav a.active { background: #0f766e; color: #fff; border-left-color: #2dd4bf; font-weight: bold; }
        .sidebar nav .menu-label {
            font-size: 10px;
            text-transform: uppercase;
            color: #5eead4;
            padding: 14px 18px 4px;
            letter-spacing: 1px;
        }
        .sidebar .footer { padding: 14px 18px; font-size: 11px; color: #5eead4; border-top: 1px solid #0f3d39; }

        /* Main content */
        .main-wrapper { margin-left: 220px; flex: 1; padding: 30px 32px; }
        .page-title { font-size: 22px; font-weight: bold; color: #1f2937; margin-bottom: 4px; }
        .page-subtitle { font-size: 13px; color: #6b7280; margin-bottom: 24px; }

        /* Alert */
        .alert-success { background: #d1fae5; color: #065f46; padding: 10px 14px; border-radius: 6px; margin-bottom: 18px; font-size: 14px; border: 1px solid #6ee7b7; }
        .alert-error   { background: #fee2e2; color: #991b1b; padding: 10px 14px; border-radius: 6px; margin-bottom: 18px; font-size: 14px; border: 1px solid #fca5a5; }

        /* Card */
        .card { background: #fff; border: 1px solid #e5e7eb; border-radius: 8px; padding: 20px 24px; margin-bottom: 24px; }
        .card-title { font-size: 15px; font-weight: bold; color: #374151; margin-bottom: 16px; padding-bottom: 10px; border-bottom: 1px solid #f3f4f6; }

        /* Form */
        .form-group { margin-bottom: 14px; }
        .form-group label { display: block; font-size: 12px; font-weight: bold; color: #374151; margin-bottom: 5px; }
        .form-group input[type=text],
        .form-group input[type=number],
        .form-group input[type=date],
        .form-group select,
        .form-group textarea {
            width: 100%; padding: 8px 10px;
            border: 1px solid #d1d5db; border-radius: 5px;
            font-size: 13px; color: #111827;
        }
        .form-group textarea { resize: vertical; }
        .btn-primary { background: #0d9488; color: #fff; border: none; padding: 9px 20px; border-radius: 5px; font-size: 13px; font-weight: bold; cursor: pointer; }
        .btn-primary:hover { background: #0f766e; }
        .btn-secondary { background: #e5e7eb; color: #374151; text-decoration: none; padding: 9px 18px; border-radius: 5px; font-size: 13px; font-weight: bold; }

        /* Table */
        table { width: 100%; border-collapse: collapse; font-size: 13px; }
        th, td { padding: 10px 14px; text-align: left; border-bottom: 1px solid #f3f4f6; }
        th { background: #f9fafb; color: #6b7280; font-weight: 600; font-size: 12px; text-transform: uppercase; letter-spacing: 0.5px; }
        tr:hover td { background: #f9fafb; }
        .aksi a { font-size: 12px; font-weight: bold; margin-right: 8px; }
    </style>
</head>
<body>

<div class="sidebar">
    <div class="brand">
        🏥 UjiBase Mini
        <span>Health Analytics</span>
    </div>
    <nav>
        <div class="menu-label">Navigasi</div>
        <a href="/ujibase_mini/" class="<?= $is_home  ? 'active' : '' ?>">Dashboard</a>

        <div class="menu-label">Master Data</div>
        <a href="/ujibase_mini/obat/"  class="<?= $is_obat  ? 'active' : '' ?>">Data Obat</a>

        <div class="menu-label">Transaksi</div>
        <a href="/ujibase_mini/resep/" class="<?= $is_resep ? 'active' : '' ?>">Data Resep</a>
    </nav>
    <div class="footer">UjiBase Mini v1.0</div>
</div>

<div class="main-wrapper">
```

> **Penting:** File `header.php` membuka tag `<html>`, `<head>`, `<body>`, dan `<div class="main-wrapper">` — **jangan ditutup di sini**. Penutupnya ada di bagian bawah setiap halaman `index.php` dan `edit.php`.

Tambahkan penutup ini di **bagian paling bawah** setiap view:

```php
</div><!-- end .main-wrapper -->
</body>
</html>
```

---

## Langkah 4: Membuat Root `index.php` (Dashboard)

Buat file `index.php` langsung di dalam folder `ujibase_mini/` — ini halaman pertama yang terbuka saat akses `http://localhost/ujibase_mini/`.

```php
<?php
require_once 'koneksi.php';
require_once 'partials/header.php';

// Hitung total data untuk statistik
$total_obat  = mysqli_fetch_row(mysqli_query($koneksi, "SELECT COUNT(*) FROM obat"))[0];
$total_resep = mysqli_fetch_row(mysqli_query($koneksi, "SELECT COUNT(*) FROM resep"))[0];
$total_stok  = mysqli_fetch_row(mysqli_query($koneksi, "SELECT SUM(stok) FROM obat"))[0] ?? 0;

// 5 resep terbaru
$query_terbaru = mysqli_query($koneksi,
    "SELECT r.tanggal_resep, r.nama_pasien, r.nama_dokter, o.nama_obat, r.jumlah
     FROM resep r
     JOIN obat o ON r.obat_id = o.id
     ORDER BY r.id DESC LIMIT 5"
);
$resep_terbaru = mysqli_fetch_all($query_terbaru, MYSQLI_ASSOC);
?>

<div class="page-title">Dashboard</div>
<div class="page-subtitle">Selamat datang di UjiBase Mini — sistem pencatatan obat dan resep</div>

<!-- Statistik Cards -->
<div style="display:grid; grid-template-columns:repeat(3,1fr); gap:16px; margin-bottom:24px;">
    <div style="background:#0d9488;color:#fff;border-radius:8px;padding:20px 22px;">
        <div style="font-size:28px;font-weight:bold;"><?= $total_obat ?></div>
        <div style="font-size:13px;margin-top:4px;opacity:0.85;">Total Jenis Obat</div>
    </div>
    <div style="background:#0369a1;color:#fff;border-radius:8px;padding:20px 22px;">
        <div style="font-size:28px;font-weight:bold;"><?= $total_resep ?></div>
        <div style="font-size:13px;margin-top:4px;opacity:0.85;">Total Data Resep</div>
    </div>
    <div style="background:#7c3aed;color:#fff;border-radius:8px;padding:20px 22px;">
        <div style="font-size:28px;font-weight:bold;"><?= number_format($total_stok, 0, ',', '.') ?></div>
        <div style="font-size:13px;margin-top:4px;opacity:0.85;">Total Stok Obat</div>
    </div>
</div>

<!-- Tabel Resep Terbaru -->
<div class="card">
    <div class="card-title">5 Resep Terbaru</div>
    <table>
        <thead>
            <tr>
                <th>Tanggal</th>
                <th>Nama Pasien</th>
                <th>Dokter</th>
                <th>Obat</th>
                <th>Jumlah</th>
            </tr>
        </thead>
        <tbody>
            <?php if (!empty($resep_terbaru)): ?>
                <?php foreach ($resep_terbaru as $row): ?>
                <tr>
                    <td><?= htmlspecialchars($row['tanggal_resep']) ?></td>
                    <td><?= htmlspecialchars($row['nama_pasien']) ?></td>
                    <td><?= htmlspecialchars($row['nama_dokter']) ?></td>
                    <td><?= htmlspecialchars($row['nama_obat']) ?></td>
                    <td><?= $row['jumlah'] ?></td>
                </tr>
                <?php endforeach; ?>
            <?php else: ?>
                <tr><td colspan="5" style="text-align:center;color:#999;padding:20px;">Belum ada data resep.</td></tr>
            <?php endif; ?>
        </tbody>
    </table>
</div>

</div><!-- end .main-wrapper -->
</body>
</html>
```

---

## Langkah 5: CRUD Modul Obat

### 5.1 — `obat/index.php` (Tampil + Form Tambah)

```php
<?php
require_once '../koneksi.php';
require_once '../partials/header.php';

// Ambil semua data obat
$query = mysqli_query($koneksi, "SELECT * FROM obat ORDER BY nama_obat ASC");
$obat  = mysqli_fetch_all($query, MYSQLI_ASSOC);
?>

<div class="page-title">Master Data Obat</div>
<div class="page-subtitle">Kelola stok dan informasi obat yang tersedia</div>

<?php if (isset($_GET['success'])): ?>
    <div class="alert-success"><?= htmlspecialchars($_GET['success']) ?></div>
<?php endif; ?>
<?php if (isset($_GET['error'])): ?>
    <div class="alert-error"><?= htmlspecialchars($_GET['error']) ?></div>
<?php endif; ?>

<div style="display:grid; grid-template-columns:1fr 2fr; gap:24px; align-items:start;">

    <!-- Form Tambah -->
    <div class="card">
        <div class="card-title">Tambah Obat Baru</div>
        <form action="store.php" method="POST">
            <div class="form-group">
                <label>Kode Obat</label>
                <input type="text" name="kode_obat" required placeholder="Cth: OBT-005">
            </div>
            <div class="form-group">
                <label>Nama Obat</label>
                <input type="text" name="nama_obat" required placeholder="Cth: Cetirizine 10mg">
            </div>
            <div class="form-group">
                <label>Harga Jual (Rp)</label>
                <input type="number" name="harga_jual" required placeholder="Cth: 8000">
            </div>
            <div class="form-group">
                <label>Stok</label>
                <input type="number" name="stok" required placeholder="Cth: 100">
            </div>
            <button type="submit" class="btn-primary">Simpan Obat</button>
        </form>
    </div>

    <!-- Tabel Daftar Obat -->
    <div class="card">
        <div class="card-title">Daftar Obat</div>
        <table>
            <thead>
                <tr>
                    <th>#</th>
                    <th>Kode</th>
                    <th>Nama Obat</th>
                    <th>Harga Jual</th>
                    <th>Stok</th>
                    <th>Aksi</th>
                </tr>
            </thead>
            <tbody>
                <?php if (!empty($obat)): ?>
                    <?php foreach ($obat as $i => $row): ?>
                    <tr>
                        <td><?= $i + 1 ?></td>
                        <td><?= htmlspecialchars($row['kode_obat']) ?></td>
                        <td><?= htmlspecialchars($row['nama_obat']) ?></td>
                        <td>Rp <?= number_format($row['harga_jual'], 0, ',', '.') ?></td>
                        <td><?= $row['stok'] ?></td>
                        <td class="aksi">
                            <a href="edit.php?id=<?= $row['id'] ?>" style="color:#2563eb;">Edit</a>
                            <a href="delete.php?id=<?= $row['id'] ?>" onclick="return confirm('Hapus obat ini?');" style="color:#dc2626;">Hapus</a>
                        </td>
                    </tr>
                    <?php endforeach; ?>
                <?php else: ?>
                    <tr><td colspan="6" style="text-align:center;color:#9ca3af;padding:20px;">Belum ada data obat.</td></tr>
                <?php endif; ?>
            </tbody>
        </table>
    </div>

</div>

</div><!-- end .main-wrapper -->
</body>
</html>
```

### 5.2 — `obat/store.php` (Proses Tambah)

```php
<?php
require_once '../koneksi.php';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $kode_obat  = mysqli_real_escape_string($koneksi, trim($_POST['kode_obat']));
    $nama_obat  = mysqli_real_escape_string($koneksi, trim($_POST['nama_obat']));
    $harga_jual = (int) $_POST['harga_jual'];
    $stok       = (int) $_POST['stok'];

    $sql = "INSERT INTO obat (kode_obat, nama_obat, harga_jual, stok)
            VALUES ('$kode_obat', '$nama_obat', $harga_jual, $stok)";

    if (mysqli_query($koneksi, $sql)) {
        header("Location: index.php?success=Data obat berhasil ditambahkan.");
    } else {
        header("Location: index.php?error=Gagal menyimpan data obat.");
    }
    exit;
}

header("Location: index.php");
exit;
```

### 5.3 — `obat/edit.php` (Form Edit)

```php
<?php
require_once '../koneksi.php';
require_once '../partials/header.php';

if (!isset($_GET['id'])) { header("Location: index.php"); exit; }

$id    = (int) $_GET['id'];
$query = mysqli_query($koneksi, "SELECT * FROM obat WHERE id = $id");
$row   = mysqli_fetch_assoc($query);

if (!$row) { header("Location: index.php"); exit; }
?>

<div class="page-title">Edit Data Obat</div>
<div class="page-subtitle">Perbarui informasi obat yang sudah ada</div>

<div class="card" style="max-width:480px;">
    <form action="update.php" method="POST">
        <input type="hidden" name="id" value="<?= $row['id'] ?>">

        <div class="form-group">
            <label>Kode Obat</label>
            <input type="text" name="kode_obat" required value="<?= htmlspecialchars($row['kode_obat']) ?>">
        </div>
        <div class="form-group">
            <label>Nama Obat</label>
            <input type="text" name="nama_obat" required value="<?= htmlspecialchars($row['nama_obat']) ?>">
        </div>
        <div class="form-group">
            <label>Harga Jual (Rp)</label>
            <input type="number" name="harga_jual" required value="<?= $row['harga_jual'] ?>">
        </div>
        <div class="form-group">
            <label>Stok</label>
            <input type="number" name="stok" required value="<?= $row['stok'] ?>">
        </div>

        <div style="display:flex; gap:10px; margin-top:16px;">
            <button type="submit" class="btn-primary">Update Data</button>
            <a href="index.php" class="btn-secondary">Batal</a>
        </div>
    </form>
</div>

</div><!-- end .main-wrapper -->
</body>
</html>
```

### 5.4 — `obat/update.php` (Proses Update)

```php
<?php
require_once '../koneksi.php';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $id         = (int) $_POST['id'];
    $kode_obat  = mysqli_real_escape_string($koneksi, trim($_POST['kode_obat']));
    $nama_obat  = mysqli_real_escape_string($koneksi, trim($_POST['nama_obat']));
    $harga_jual = (int) $_POST['harga_jual'];
    $stok       = (int) $_POST['stok'];

    $sql = "UPDATE obat SET
                kode_obat  = '$kode_obat',
                nama_obat  = '$nama_obat',
                harga_jual = $harga_jual,
                stok       = $stok
            WHERE id = $id";

    if (mysqli_query($koneksi, $sql)) {
        header("Location: index.php?success=Data obat berhasil diperbarui.");
    } else {
        header("Location: index.php?error=Gagal memperbarui data obat.");
    }
    exit;
}

header("Location: index.php");
exit;
```

### 5.5 — `obat/delete.php` (Proses Hapus)

```php
<?php
require_once '../koneksi.php';

if (isset($_GET['id'])) {
    $id  = (int) $_GET['id'];
    $sql = "DELETE FROM obat WHERE id = $id";

    if (mysqli_query($koneksi, $sql)) {
        header("Location: index.php?success=Data obat berhasil dihapus.");
    } else {
        // Dipicu oleh FK ON DELETE RESTRICT jika obat sudah dipakai di resep
        header("Location: index.php?error=Gagal dihapus! Obat ini sudah digunakan dalam data resep.");
    }
    exit;
}

header("Location: index.php");
exit;
```

---

## Langkah 6: CRUD Modul Resep

### 6.1 — `resep/index.php` (Tampil + Form Tambah)

```php
<?php
require_once '../koneksi.php';
require_once '../partials/header.php';

// Ambil semua resep dengan JOIN ke tabel obat
$query_resep = mysqli_query($koneksi,
    "SELECT r.*, o.nama_obat, o.kode_obat
     FROM resep r
     JOIN obat o ON r.obat_id = o.id
     ORDER BY r.tanggal_resep DESC"
);
$resep = mysqli_fetch_all($query_resep, MYSQLI_ASSOC);

// Ambil daftar obat untuk dropdown
$query_obat = mysqli_query($koneksi, "SELECT id, kode_obat, nama_obat FROM obat ORDER BY nama_obat ASC");
$list_obat  = mysqli_fetch_all($query_obat, MYSQLI_ASSOC);
?>

<div class="page-title">Data Resep Medis</div>
<div class="page-subtitle">Pencatatan resep pasien berdasarkan obat yang tersedia</div>

<?php if (isset($_GET['success'])): ?>
    <div class="alert-success"><?= htmlspecialchars($_GET['success']) ?></div>
<?php endif; ?>
<?php if (isset($_GET['error'])): ?>
    <div class="alert-error"><?= htmlspecialchars($_GET['error']) ?></div>
<?php endif; ?>

<div style="display:grid; grid-template-columns:1fr 2fr; gap:24px; align-items:start;">

    <!-- Form Tambah -->
    <div class="card">
        <div class="card-title">Tambah Resep Baru</div>
        <form action="store.php" method="POST">
            <div class="form-group">
                <label>Tanggal Resep</label>
                <input type="date" name="tanggal_resep" required value="<?= date('Y-m-d') ?>">
            </div>
            <div class="form-group">
                <label>Nama Pasien</label>
                <input type="text" name="nama_pasien" required placeholder="Cth: Budi Raharjo">
            </div>
            <div class="form-group">
                <label>Nama Dokter</label>
                <input type="text" name="nama_dokter" required placeholder="Cth: dr. Sari Dewi">
            </div>
            <div class="form-group">
                <label>Pilih Obat</label>
                <select name="obat_id" required>
                    <option value="">-- Pilih Obat --</option>
                    <?php foreach ($list_obat as $obat): ?>
                        <option value="<?= $obat['id'] ?>"><?= htmlspecialchars($obat['kode_obat'] . ' - ' . $obat['nama_obat']) ?></option>
                    <?php endforeach; ?>
                </select>
            </div>
            <div class="form-group">
                <label>Jumlah</label>
                <input type="number" name="jumlah" required placeholder="Cth: 10" min="1">
            </div>
            <div class="form-group">
                <label>Keterangan</label>
                <textarea name="keterangan" rows="2" placeholder="Cth: Diminum 3x sehari sesudah makan"></textarea>
            </div>
            <button type="submit" class="btn-primary">Simpan Resep</button>
        </form>
    </div>

    <!-- Tabel Daftar Resep -->
    <div class="card">
        <div class="card-title">Daftar Resep</div>
        <table>
            <thead>
                <tr>
                    <th>#</th>
                    <th>Tanggal</th>
                    <th>Pasien</th>
                    <th>Dokter</th>
                    <th>Obat</th>
                    <th>Jml</th>
                    <th>Keterangan</th>
                    <th>Aksi</th>
                </tr>
            </thead>
            <tbody>
                <?php if (!empty($resep)): ?>
                    <?php foreach ($resep as $i => $row): ?>
                    <tr>
                        <td><?= $i + 1 ?></td>
                        <td><?= htmlspecialchars($row['tanggal_resep']) ?></td>
                        <td><?= htmlspecialchars($row['nama_pasien']) ?></td>
                        <td><?= htmlspecialchars($row['nama_dokter']) ?></td>
                        <td><?= htmlspecialchars($row['kode_obat'] . ' - ' . $row['nama_obat']) ?></td>
                        <td><?= $row['jumlah'] ?></td>
                        <td><?= htmlspecialchars($row['keterangan']) ?></td>
                        <td class="aksi">
                            <a href="edit.php?id=<?= $row['id'] ?>" style="color:#2563eb;">Edit</a>
                            <a href="delete.php?id=<?= $row['id'] ?>" onclick="return confirm('Hapus resep ini?');" style="color:#dc2626;">Hapus</a>
                        </td>
                    </tr>
                    <?php endforeach; ?>
                <?php else: ?>
                    <tr><td colspan="8" style="text-align:center;color:#9ca3af;padding:20px;">Belum ada data resep.</td></tr>
                <?php endif; ?>
            </tbody>
        </table>
    </div>

</div>

</div><!-- end .main-wrapper -->
</body>
</html>
```

### 6.2 — `resep/store.php` (Proses Tambah)

```php
<?php
require_once '../koneksi.php';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $tanggal_resep = mysqli_real_escape_string($koneksi, $_POST['tanggal_resep']);
    $nama_pasien   = mysqli_real_escape_string($koneksi, trim($_POST['nama_pasien']));
    $nama_dokter   = mysqli_real_escape_string($koneksi, trim($_POST['nama_dokter']));
    $obat_id       = (int) $_POST['obat_id'];
    $jumlah        = (int) $_POST['jumlah'];
    $keterangan    = mysqli_real_escape_string($koneksi, trim($_POST['keterangan']));

    $sql = "INSERT INTO resep (tanggal_resep, nama_pasien, nama_dokter, obat_id, jumlah, keterangan)
            VALUES ('$tanggal_resep', '$nama_pasien', '$nama_dokter', $obat_id, $jumlah, '$keterangan')";

    if (mysqli_query($koneksi, $sql)) {
        header("Location: index.php?success=Data resep berhasil ditambahkan.");
    } else {
        header("Location: index.php?error=Gagal menyimpan data resep.");
    }
    exit;
}

header("Location: index.php");
exit;
```

### 6.3 — `resep/edit.php` (Form Edit)

```php
<?php
require_once '../koneksi.php';
require_once '../partials/header.php';

if (!isset($_GET['id'])) { header("Location: index.php"); exit; }

$id    = (int) $_GET['id'];
$query = mysqli_query($koneksi, "SELECT * FROM resep WHERE id = $id");
$row   = mysqli_fetch_assoc($query);

if (!$row) { header("Location: index.php"); exit; }

// Ambil daftar obat untuk dropdown
$query_obat = mysqli_query($koneksi, "SELECT id, kode_obat, nama_obat FROM obat ORDER BY nama_obat ASC");
$list_obat  = mysqli_fetch_all($query_obat, MYSQLI_ASSOC);
?>

<div class="page-title">Edit Data Resep</div>
<div class="page-subtitle">Perbarui data resep yang sudah ada</div>

<div class="card" style="max-width:520px;">
    <form action="update.php" method="POST">
        <input type="hidden" name="id" value="<?= $row['id'] ?>">

        <div class="form-group">
            <label>Tanggal Resep</label>
            <input type="date" name="tanggal_resep" required value="<?= htmlspecialchars($row['tanggal_resep']) ?>">
        </div>
        <div class="form-group">
            <label>Nama Pasien</label>
            <input type="text" name="nama_pasien" required value="<?= htmlspecialchars($row['nama_pasien']) ?>">
        </div>
        <div class="form-group">
            <label>Nama Dokter</label>
            <input type="text" name="nama_dokter" required value="<?= htmlspecialchars($row['nama_dokter']) ?>">
        </div>
        <div class="form-group">
            <label>Pilih Obat</label>
            <select name="obat_id" required>
                <?php foreach ($list_obat as $obat): ?>
                    <option value="<?= $obat['id'] ?>" <?= ($row['obat_id'] == $obat['id']) ? 'selected' : '' ?>>
                        <?= htmlspecialchars($obat['kode_obat'] . ' - ' . $obat['nama_obat']) ?>
                    </option>
                <?php endforeach; ?>
            </select>
        </div>
        <div class="form-group">
            <label>Jumlah</label>
            <input type="number" name="jumlah" required value="<?= $row['jumlah'] ?>" min="1">
        </div>
        <div class="form-group">
            <label>Keterangan</label>
            <textarea name="keterangan" rows="2"><?= htmlspecialchars($row['keterangan']) ?></textarea>
        </div>

        <div style="display:flex; gap:10px; margin-top:16px;">
            <button type="submit" class="btn-primary">Update Resep</button>
            <a href="index.php" class="btn-secondary">Batal</a>
        </div>
    </form>
</div>

</div><!-- end .main-wrapper -->
</body>
</html>
```

### 6.4 — `resep/update.php` (Proses Update)

```php
<?php
require_once '../koneksi.php';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $id            = (int) $_POST['id'];
    $tanggal_resep = mysqli_real_escape_string($koneksi, $_POST['tanggal_resep']);
    $nama_pasien   = mysqli_real_escape_string($koneksi, trim($_POST['nama_pasien']));
    $nama_dokter   = mysqli_real_escape_string($koneksi, trim($_POST['nama_dokter']));
    $obat_id       = (int) $_POST['obat_id'];
    $jumlah        = (int) $_POST['jumlah'];
    $keterangan    = mysqli_real_escape_string($koneksi, trim($_POST['keterangan']));

    $sql = "UPDATE resep SET
                tanggal_resep = '$tanggal_resep',
                nama_pasien   = '$nama_pasien',
                nama_dokter   = '$nama_dokter',
                obat_id       = $obat_id,
                jumlah        = $jumlah,
                keterangan    = '$keterangan'
            WHERE id = $id";

    if (mysqli_query($koneksi, $sql)) {
        header("Location: index.php?success=Data resep berhasil diperbarui.");
    } else {
        header("Location: index.php?error=Gagal memperbarui data resep.");
    }
    exit;
}

header("Location: index.php");
exit;
```

### 6.5 — `resep/delete.php` (Proses Hapus)

```php
<?php
require_once '../koneksi.php';

if (isset($_GET['id'])) {
    $id  = (int) $_GET['id'];
    $sql = "DELETE FROM resep WHERE id = $id";

    if (mysqli_query($koneksi, $sql)) {
        header("Location: index.php?success=Data resep berhasil dihapus.");
    } else {
        header("Location: index.php?error=Gagal menghapus data resep.");
    }
    exit;
}

header("Location: index.php");
exit;
```

---

## ✅ Misi Selesai!

Ringkasan seluruh file yang dibuat:

| No  | Path File             | Fungsi                            |
| --- | --------------------- | --------------------------------- |
| 1   | `index.php`           | Dashboard utama (landing page)    |
| 2   | `koneksi.php`         | Koneksi ke database               |
| 3   | `partials/header.php` | Sidebar navigasi + layout wrapper |
| 4   | `obat/index.php`      | Tampil daftar + form tambah obat  |
| 5   | `obat/store.php`      | Proses simpan obat baru           |
| 6   | `obat/edit.php`       | Form edit obat                    |
| 7   | `obat/update.php`     | Proses update obat                |
| 8   | `obat/delete.php`     | Proses hapus obat                 |
| 9   | `resep/index.php`     | Tampil daftar + form tambah resep |
| 10  | `resep/store.php`     | Proses simpan resep baru          |
| 11  | `resep/edit.php`      | Form edit resep                   |
| 12  | `resep/update.php`    | Proses update resep               |
| 13  | `resep/delete.php`    | Proses hapus resep                |

> **Akses di browser:** `http://localhost/ujibase_mini/` — sidebar akan mengarahkan ke semua modul.
