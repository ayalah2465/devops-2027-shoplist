# שיעור 6 — תשובות

## 1. למה shoplist-backend:local ולא docker-backend? ומה בלי imagePullPolicy: Never?
השם חייב להיות זהה לשם שה-build נתן לתמונה, אחרת ה-Deployment מחפש תמונה שלא קיימת. imagePullPolicy: Never מחייב שימוש בתמונה המקומית בלבד; בלי זה kubelet היה מנסה למשוך את shoplist-backend:local מ-Docker Hub, נכשל, והפוד היה נתקע ב-ErrImagePull.

## 2. פענוח הסיסמה מה-Secret
kubectl get secret postgres-secret -n shoplist -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
base64 אינו הצפנה אלא קידוד — כל מי שיש לו הרשאת קריאה ל-Secret מקבל את הסיסמה בטקסט גלוי. Secret מפריד את הסוד מהקוד ושולט בהרשאות, אך אינו מגן מפני בעל הרשאה.

## 3. מה אחראי לשרידות הנתונים, ואיזו פקודה הייתה מוחקת אותם?
postgres-pvc.yml — ה-PersistentVolumeClaim. הפוד נמחק אך הנתונים נשארים על ה-PVC ומתחברים מחדש. מחיקה: kubectl delete pvc postgres-pvc -n shoplist.

## 4. מה קורה אם ה-backend עולה לפני postgres?
בלי depends_on אין סדר עלייה. ה-backend ינסה להתחבר, ייכשל וייפל ויעלה שוב (Backoff) עד שה-DB יהיה זמין. ה-readinessProbe ל-/health מונע ממנו לקבל תנועה לפני שהוא מוכן, וה-Service מנתב בקשות רק לפודים Ready.

## 5. ד8 — מה נראה ומה קרה?
ה-selector שונה ל-app: backnd, ולכן ה-Service לא מצא פודים תואמים. ה-pods של backend לא נפלו — הם המשיכו לרוץ, כי הבעיה ב-Service ולא ב-Deployment. get endpoints החזיר <none>, ו-describe הראה Endpoints: <none>. הדפדפן הציג שגיאת חיבור כי ה-frontend לא מצא כתובת לפנות אליה. לאחר החזרת ה-selector — הנוריות חזרו לירוק.

## 6. שינוי ב-ConfigMap לא מתעדכן בפוד רץ. למה, ומה הפתרון?
משתני סביבה נקראים פעם אחת, בזמן עליית ה-container. פוד קיים ממשיך להחזיק את הערך הישן. הפתרון: kubectl rollout restart deployment/backend -n shoplist.

## 7. כמה פעמים כתבתן shoplist, וכמה קבצים לשינוי לעותק שני?
"shoplist" מופיע בכל 12 הקבצים — פעם כ-namespace ופעם בשם המשאב. להרצת עותק ב-namespace בשם shoplist-test יש לשנות את כל 12 הקבצים (11 חוץ מ-namespace.yml). זו הבעיה שה-Release.Namespace ב-Helm פותר.