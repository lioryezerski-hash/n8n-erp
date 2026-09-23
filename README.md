# n8n ERP

מערכת ERP מבוססת n8n: חשבוניות, לידים, מוצרים ומשימות ב-Airtable, סוכני טלגרם (לקוחות + מנהל), RAG על מדיניות ומוצרים, ו-dashboard שמדבר רק עם WF13.

> ⚠️ שמור את ה-repo הזה **Private**. אל תעלה אליו מפתחות API, טוקנים או credentials.

## מבנה

| תיקייה | תוכן |
|---|---|
| `workflows/` | קובצי ה-JSON של ה-workflows, מוכנים לייבוא |
| `dashboard/index.html` | ה-frontend. פונה רק ל-`/webhook/erp` (WF13) |

## Workflows

| קובץ | מה הוא עושה | Credentials |
|---|---|---|
| WF01 – Invoice Validation | חשבונית חדשה ב-Airtable: בדיקה, חישוב מע"מ 18% וסה"כ | Airtable |
| WF02 – Lead Intake | ליד חדש: בדיקת תקינות וסימון כפילויות לפי אימייל | Airtable |
| WF03 – Sales Cold Email | כל 3 שעות: מייל פנייה אחד לליד בסטטוס New | Airtable, OpenAI, Gmail |
| WF04 – Sales Reply Checking | כל 30 דקות: מזהה תשובות ומסמן Replied | Airtable, Gmail |
| WF05 – Customer Service Agent | בוט טלגרם ללקוחות עם RAG | OpenAI, Telegram Customer Service Bot |
| WF06 – Load Policy | טוען את מסמך המדיניות ל-RAG (ידני) | OpenAI |
| WF07 – Load Products | טוען את טבלת המוצרים ל-RAG (ידני, אחרי WF06) | Airtable, OpenAI |
| WF08 – Invoice Document Generation | כל דקה: מפיק HTML לחשבוניות Validated ומעלה ל-Drive | Airtable, Google Drive |
| WF09 – Manager Agent | בוט טלגרם למנהל בלבד (Chat ID `480500793`). המדדים מחושבים ב-n8n, ו-OpenAI רק מנסח את התשובה | Airtable, OpenAI, Telegram Manager Bot |
| WF13 – Application Backend | ה-webhook `POST /webhook/erp` של ה-dashboard | Airtable, OpenAI |

> WF09 נבנה מחדש (הקובץ המקורי לא נמצא). כדאי להשוות אותו לגרסה שבחשבון n8n, אם קיימת.

## פרטי מערכת

- Airtable base: `app0bWNSJs9TYcFmM` (Invoices, Leads, Products, Tasks)
- תיקיית Google Drive לחשבוניות: `1mEg_12f9bGRKmsXTSrpdOD9NVg_H_-sc`
- WF13: webhook `POST /webhook/erp`, ה-backend של ה-dashboard
- Telegram Owner Chat ID: `480500793` (רק הוא מורשה להשתמש ב-Manager Agent)
- מע"מ: 18% (`VAT_RATE = 0.18` ב-WF01). מחושב ב-n8n, לא ב-OpenAI
- ה-Airtable base ותיקיית ה-Drive כבר מוגדרים בקבצים

## התקנה בחשבון n8n חדש

1. **ייבוא:** Overview → Create → Import from file, לכל קובץ ב-`workflows/`.
2. **Credentials** (Settings → Credentials → Add Credential):
   - Airtable Personal Access Token account
   - OpenAI account
   - Telegram Customer Service Bot
   - Telegram Manager Bot
   - Gmail OAuth2 API
   - Google Drive OAuth2 API
3. **קישור:** בקבצים ה-credentials מופיעים לפי שם בלבד (בלי מזהים). אם נשאר node עם ⚠️, בוחרים בו את ה-credential המתאים ושומרים. ל-Gmail ול-Drive: לתת ל-credentials את השמות `Gmail account` ו-`Google Drive account`.
4. **סיסמת WF13:** ב-WF13, ב-node בשם **Check ERP Key**, מחליפים את `REPLACE_WITH_DASHBOARD_PASSWORD` בסיסמת ה-dashboard.
5. **כיבוי החשבון הישן:** לפני ההפעלה, מכבים את כל ה-workflows בחשבון הקודם. לכל בוט טלגרם יכול להיות webhook אחד בלבד.
6. **הפעלה:** מפעילים את כל ה-workflows (toggle ב-Overview).
7. **אתחול RAG, לפי הסדר:** מריצים את WF6 (Load Policy) ומחכים ל-Succeeded, ורק אז מריצים את WF7 (Load Products). ה-RAG נשמר בזיכרון של n8n (In-Memory Vector Store), ולכן אחרי כל restart של ה-instance צריך להריץ שוב את WF06 ואחריו את WF07.
8. **Dashboard:** אם כתובת ה-instance השתנתה, מעדכנים את `WEBHOOK` ב-`dashboard/index.html`.

## Dashboard

`dashboard/index.html` מדבר רק עם WF13, ואף פעם לא ישירות עם Airtable או OpenAI. השדות בטופס תואמים למה שה-workflows מצפים לקבל:

| טבלה | שדות |
|---|---|
| Invoices | `InvoiceNumber`, `CustomerId`, `Amount`. את `VatAmount`, `Total` ו-`Status` ממלא WF01 |
| Leads | `Name`, `Email`, `Company`. את `Status` ממלאים WF02, WF03 ו-WF04 |
| Products | `Name`, `Category`, `Price`, `Description` |

אם שם של עמודה ב-Airtable שונה, צריך לעדכן אותו ב-`TABLE_FIELDS` בקובץ.

## אבטחת WF13

ה-dashboard שולח את הסיסמה ב-header בשם `X-ERP-Key`. ב-WF13, ה-node **Check ERP Key** בודק אותה לפני כל גישה ל-Airtable או ל-OpenAI. בקשה בלי סיסמה נכונה מקבלת 401.

**החלפת סיסמה:**
1. מחשבים SHA-256 לסיסמה החדשה: `echo -n "NEW" | sha256sum`.
2. מעדכנים את `PW_SHA256` ב-`dashboard/index.html`.
3. מעדכנים את הערך ב-node **Check ERP Key** ב-WF13.

לעולם לא לשמור את הסיסמה האמיתית בקבצים שב-repo.
