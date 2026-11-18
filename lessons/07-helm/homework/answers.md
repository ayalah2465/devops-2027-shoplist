# שיעור 7 — Helm: שאלות

## 1. ההבדל בין chart ל-release

chart הוא התבנית: תיקייה עם `templates/` ו-`values.yaml`, שמתארת את כל משאבי האפליקציה בלי ערכים קשיחים. release הוא התקנה אחת של אותו chart, עם שם וב-namespace מסוים. בשיעור הותקן release בשם `shop` ב-namespace `shop` מתוך ה-chart `shoplist` — chart אחד, והתקנה אחת. מאותו chart אפשר להתקין כמה releases במקביל, כל אחד עם ערכים משלו ובנפרד.

## 2. למה הוסר namespace.yml, ומה מחליף אותו

ה-namespace אינו חלק מהתבנית, אלא בחירה של כל התקנה בנפרד — אותו chart יכול לשבת ב-`shop`, ב-`shop2` או בכל namespace אחר. לכן `namespace.yml` הוסר מה-chart, וכל שאר הקבצים משתמשים ב-`{{ .Release.Namespace }}`. את ה-namespace מספקים בזמן ההתקנה עם `-n shop`, ואם הוא אינו קיים — `--create-namespace` יוצר אותו.

## 3. מאיפה הגיעו שאר הערכים ב-values-big.yaml

`values-big.yaml` מכיל רק את מה ששונה מברירת המחדל: `backend.replicas: 3` ו-`stats.replicas: 2`. כל השאר — ה-images, ה-`nodePort`, הסיסמה וגודל האחסון — נלקח מ-`values.yaml` שבתוך ה-chart. Helm ממזג את הקבצים: ברירת המחדל קודם, אחריה קובצי `-f` לפי הסדר, ו-`--set` אחרון. בפועל ה-upgrade יצר שלושה pods של backend ושניים של stats, ואת השאר לקח כברירת מחדל.

## 4. למה האפליקציה עבדה כשה-image היה שבור

בגלל ה-`readinessProbe` שהוגדר בכל ה-Deployments. Kubernetes מעלה pod חדש עם ה-image השבור (`backend.image.tag=oops`), אך אי אפשר להעביר אליו תנועה כל עוד הוא אינו עובר את בדיקת המוכנות. לכן שלושת ה-pods הישנים — עם ה-image התקין — לא הוסרו, והשירות ב-30080 המשיך לענות ולהציג את שלושת המוצרים. ה-pod השבור נראה ב-`ContainerCreating` ולאחר מכן ב-`ErrImageNeverPull`, אך לא פגע בזמינות.

## 5. למה יש 4 revisions ולא 2 אחרי rollback

ההיסטוריה ב-Helm אינה נמחקת ואינה "מתקפלת". ה-`rollback` אינו מוחק את revisions 3, אלא יוצר revision חדש — מספר 4 — שתוכנו זהה לתוכן של revision 2. כך נשמר תיעוד מלא: מה הותקן, מה נשבר, ומתי בוצעה החזרה. ה-revision החדש מסומן כ-`Rollback to 2`.

## 6. מה היה קורה למוצרים בלי helm.sh/resource-policy: keep

ה-`helm uninstall` מוחק כל משאב שנוצר על ידי ה-release — כולל ה-PVC. בלי ה-annotation, ה-PVC היה נמחק יחד עם כל הנתונים שבו, וההתקנה מחדש הייתה יוצרת מסד נתונים ריק. עם ה-annotation, ה-PVC נותר `Bound` לאחר ה-uninstall (הופיעה ההודעה `These resources were kept due to the resource policy`), ושלושת המוצרים — לחם, חלב וביצים — הופיעו מיד לאחר ההתקנה מחדש.

## 7. מה קרה עם {{ .Values.backend.replica }}

Helm לא מפיק שגיאה על מפתח שאינו קיים, אלא מחליף את החור בערך ריק. בפועל השורה `replicas:` נשארה בלי ערך, ו-Kubernetes השתמש בברירת המחדל שלו (replica אחת) — בלי שאף אחד שם לב. הדרך לגלות: `helm template` לפני ההתקנה והסתכלות על הפלט. `helm lint` לא בהכרח מזהה כתיב שגוי כזה, ולכן ההשוואה ל-`values.yaml` היא הבדיקה הקובעת.