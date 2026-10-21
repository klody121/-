# نظام إدارة تأجير المعدات والأجهزة

**إعداد: مشعل حسن**

مهمة الأسبوع: الاتصال بقاعدة البيانات، إنشاء حساب، تسجيل الدخول، وإنهاء الجلسات باستخدام PHP وMySQL.

## التجهيز

ضع ملفات PHP التالية في مجلد المشروع نفسه. نفّذ SQL التالي في MySQL، وعدّل بيانات الاتصال حسب بيئتك. صفحة `dashboard.php` يوفرها الفريق.

```sql
CREATE DATABASE IF NOT EXISTS equipment_rental CHARACTER SET utf8mb4;
USE equipment_rental;
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL
);
```

## 1. الاتصال — db.php

```php
<?php
mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT);
try {
    $conn = new mysqli('localhost', 'root', '', 'equipment_rental');
    $conn->set_charset('utf8mb4');
} catch (mysqli_sql_exception $e) {
    exit('تعذر الاتصال بقاعدة البيانات');
}
```

## 2. إنشاء حساب — register.php

```php
<?php
require_once __DIR__ . '/db.php';
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $user = trim($_POST['username'] ?? '');
    $pass = $_POST['password'] ?? '';
    if ($user === '' || strlen($user) > 100 ||
        strlen($pass) < 8 || strlen($pass) > 72) {
        exit('أدخل اسمًا صحيحًا وكلمة مرور بين 8 و72 بايت');
    }
    $hash = password_hash($pass, PASSWORD_DEFAULT);
    $stmt = $conn->prepare(
        'INSERT INTO users (username, password) VALUES (?, ?)'
    );
    $stmt->bind_param('ss', $user, $hash);
    try {
        $stmt->execute();
        echo 'تم إنشاء الحساب';
    } catch (mysqli_sql_exception $e) {
        echo 'تعذر التسجيل، قد يكون الاسم مستخدمًا';
    }
}
?>
<form method="post">
    <input name="username" placeholder="اسم المستخدم" required>
    <input type="password" name="password"
           placeholder="كلمة المرور" minlength="8" required>
    <button type="submit">إنشاء حساب</button>
</form>
```

## 3. تسجيل الدخول — login.php

```php
<?php
session_start();
require_once __DIR__ . '/db.php';
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $user = trim($_POST['username'] ?? '');
    $pass = $_POST['password'] ?? '';
    $stmt = $conn->prepare(
        'SELECT id, password FROM users WHERE username = ?'
    );
    $stmt->bind_param('s', $user);
    $stmt->execute();
    $stmt->bind_result($id, $hash);
    if ($stmt->fetch() && password_verify($pass, $hash)) {
        session_regenerate_id(true);
        $_SESSION['user_id'] = $id;
        header('Location: dashboard.php');
        exit;
    }
    echo 'بيانات الدخول غير صحيحة';
}
?>
<form method="post">
    <input name="username" placeholder="اسم المستخدم" required>
    <input type="password" name="password"
           placeholder="كلمة المرور" required>
    <button type="submit">دخول</button>
</form>
```

## 4. إنهاء الجلسة — logout.php

```php
<?php
session_start();
$_SESSION = [];
if (ini_get('session.use_cookies')) {
    $p = session_get_cookie_params();
    setcookie(session_name(), '', time() - 3600,
        $p['path'], $p['domain'], $p['secure'], $p['httponly']);
}
session_destroy();
header('Location: login.php');
exit;
```

## حماية صفحات النظام

ضع المقطع التالي في بداية الصفحة المحمية قبل أي HTML:

```php
<?php
session_start();
if (!isset($_SESSION['user_id'])) {
    header('Location: login.php');
    exit;
}
```

## التحقق

أنشئ حسابًا من `register.php`، ثم ادخل من `login.php`. افتح `logout.php` ثم تحقق أن الصفحة المحمية تعيدك إلى تسجيل الدخول.

تستخدم الأكواد تجزئة كلمات المرور والاستعلامات المحضرة. هذه نسخة تعليمية مختصرة؛ لم تُختبر على بيئة المشروع، ولا تتضمن حماية CSRF أو تحديد محاولات الدخول.
