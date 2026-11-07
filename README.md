# WordPress Compromise — Incident Report

> **تحقيق في اختراق موقع ووردبريس عبر ثغرة RCE في إضافة Simple File List**

![Status](https://img.shields.io/badge/status-closed-red)
![Severity](https://img.shields.io/badge/severity-critical-red)
![Date](https://img.shields.io/badge/date-2021--01--14-blue)
![WordPress](https://img.shields.io/badge/WordPress-5.6-21759b)

---

## 📋 ملخص تنفيذي

في **14 يناير 2021**، تعرّض موقع ووردبريس (الإصدار 5.6) لاختراق كامل عبر استغلال
ثغرة **رفع ملفات غير مقيّد** (Unrestricted File Upload) في إضافة **Simple File List**،
مما سمح للمهاجم برفع **webshell** (`fr34k.php`) وتنفيذ أوامر على نظام التشغيل
(Remote Code Execution — RCE).

| البند | القيمة |
|-------|--------|
| **تاريخ الحادثة** | 2021-01-14 |
| **مدة الهجوم** | ~48 دقيقة (05:42 → 06:30 UTC) |
| **الثغرة المستغلة** | Simple File List — Unrestricted File Upload → RCE |
| **الأثر** | تنفيذ أوامر على الخادم + احتمال تسريب قاعدة البيانات |
| **عدد المهاجمين** | 3 عناوين IP منسّقة |
| **الحالة** | ✅ تم الاحتواء (افتراضيًا) |

---

## 🕐 الجدول الزمني للهجوم

| # | الوقت (UTC) | الحدث | IP المهاجم | الدليل |
|---|-------------|-------|-----------|--------|
| 1 | 05:42:34 | استطلاع أولي (`GET /`) | 119.241.22.121 | `GET /` |
| 2 | 05:42:47 | فحص WordPress REST API | — | `GET /wp-json/wp/v2/types/post` |
| 3 | 05:43:26 | فحص `xmlrpc.php`, `wp-json`, `author/admin` | 119.241.22.121 | بصمة WPScan |
| 4 | 05:43:28 | محاولة XSS في `?s=` | 119.241.22.121 | `?s=p@y<"'p@y` |
| 5 | 05:47:07 | نشر تعليق spam | 119.241.22.121 | `POST /wp-comments-post.php` |
| 6 | 05:49:55 | فحص مسارات إدارية (>50 مسار) | 119.241.22.121 | `/admin2/`, `/bb-admin/`... |
| 7 | 05:51:14 | بدء Brute Force | 197.23.128.35 | `POST /wp-login.php` |
| 8 | 05:51:40 | تفعيل الحظر (iThemes) | — | `403` + Lockout Notification |
| 9 | 05:56–06:00 | تجاوز الحظر بـ IPs متعددة | 197.13.28.x | 8 عناوين مختلفة |
| 10 | **06:01:41** | **WPScan مكتشف** | 119.241.22.121 | `"WPScan v3.8.10"` |
| 11 | 06:03:41 | فحص ملفات حساسة | 168.22.54.119 | `dump.sql`, `debug.log` |
| 12 | 06:04:18 | بحث عن نسخ `wp-config` | 168.22.54.119 | `wp-config.php.bak`... |
| 13 | 06:08–06:16 | حصر الإضافات المثبتة | 103.212.94.19 | `simple-file-list` = 200 |
| 14 | **06:12:53** | **sqlmap مكتشف** | 168.22.54.119 | `"sqlmap/1.4.11"` |
| 15 | **06:14:00** | **حقن SQL لقراءة `/etc/passwd`** | 168.22.54.119 | `UNION SELECT ... xp_cmdshell` |
| 16 | **06:26:53** | **🔴 رفع webshell** | 119.241.22.121 | `ee-upload-engine.php` + `fr34k.png` |
| 17 | **06:27:04** | **تأكيد وجود shell** | 103.69.55.212 | `fr34k.php` → 200 |
| 18 | **06:27:06** | **🔴 أول تنفيذ أمر** | 103.69.55.212 | `POST fr34k.php` |
| 19 | 06:27–06:30 | تنفيذ أوامر متعددة | 103.69.55.212 | أحجام استجابة متغيرة |
| 20 | 06:30:05 | حذف الـ shell (تنظيف) | 103.69.55.212 | `fr34k.php` → 404 |
| 21 | 06:32:58 | ظهور phpMyAdmin 5.0.4 | 116.23.212.69 | `js/whitelist.php?v=5.0.4` |
| 22 | 07:46:52 | ملف غامض | 172.21.0.1 | `/YouShouldSeeThis.txt` |

---

## 🎯 مؤشرات الاختراق (Indicators of Compromise)

### عناوين IP المهاجمين

| IP | الدور | النطاق |
|----|-------|--------|
| `119.241.22.121` | المهاجم الرئيسي (استطلاع + رفع shell) | اليابان — `mesh.ad.jp` |
| `168.22.54.119` | حقن SQL + sqlmap | — |
| `103.69.55.212` | مشغّل الـ webshell | — |
| `103.212.94.19` | فحص الإضافات | — |
| `197.23.128.35` | Brute force | — |
| `197.13.28.x` | تجاوز الحظر (IP rotation) | — |

### الملفات الخبيثة
wp-content/uploads/simple-file-list/fr34k.png ← webshell مموّه (84,690 بايت)
wp-content/uploads/simple-file-list/fr34k.php ← webshell منفّذ (تم حذفه لاحقًا)


### الأدوات المستخدمة من المهاجم

| الأداة | الدليل |
|--------|--------|
| `WPScan v3.8.10` | `"WPScan v3.8.10 (https://wpscan.org/)"` |
| `sqlmap/1.4.11` | `"sqlmap/1.4.11#stable (http://sqlmap.org)"` |
| `python-requests/2.24.0` | سكربت استغلال آلي |
| `Mozilla/4.0 ... MSIE 6.0` | User-Agent مزيّف (أداة اختراق) |

### مسارات مشبوهة تم طلبها
/wp-content/plugins/simple-file-list/ee-upload-engine.php
/wp-content/plugins/simple-file-list/ee-file-engine.php
/wp-content/uploads/simple-file-list/fr34k.php
/wp-login.php?itsec-hb-token=adminlogin&mglS=2151+AND+1=1+UNION...
/wp-config.php.bak | .swp | .old | .save | .txt | .zip
/wp-content/uploads/dump.sql
/wp-content/debug.log

---

## 🔬 التحليل التقني

### كيف حدث الاختراق؟

1. **الاستطلاع (05:42–06:01):** المهاجم استخدم WPScan لتحديد إصدار ووردبريس والإضافات المثبتة.
2. **حصر الإضافات (06:08–06:16):** طلب مسارات كل إضافة لتحديد أيها موجود.
3. **رفع الـ webshell (06:26:53):**
   - `POST /ee-upload-engine.php` → رفع `fr34k.png` (84KB).
   - `POST /ee-file-engine.php` → تحويل/إعادة تسمية إلى `fr34k.php`.
4. **التنفيذ (06:27:06):** `POST /fr34k.php` → تنفيذ أوامر على الخادم.
5. **التنظيف (06:30:05):** حذف `fr34k.php` لمحو الأدلة.
6. **الوصول لقاعدة البيانات (06:32):** ظهور phpMyAdmin 5.0.4.

### لماذا نجح الهجوم؟

| الثغرة | الوصف |
|--------|-------|
| **Simple File List (RCE)** | رفع ملفات بدون تقييد الامتدادات → webshell |
| **phpMyAdmin 5.0.4 مكشوف** | وصول مباشر لقاعدة البيانات (CVE-2020-26934/26935) |
| **xmlrpc.php مفتوح** | يُستخدم لـ brute force |
| **iThemes Security ضعيف** | حظر IP واحد فقط — تجاوزه المهاجم بـ IP rotation |
| **كلمات مرور ضعيفة** | محاولات brute force استمرت دون حظر شامل |

---

