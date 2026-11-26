# שיעור 6 — תשובות

**1. למה `shoplist-backend:local`, ומה בלי `imagePullPolicy: Never`?**
ה-Deployment מחפש image לפי השם שכתוב בקובץ, ולכן בנינו את ה-images בדיוק בשמות האלה (`docker-backend` היה השם ש-Compose נתן, והוא לא מופיע בקבצים). בלי `imagePullPolicy: Never`, אם ה-cluster לא מוצא את ה-image אצלו (למשל ב-node אחר, או ב-cluster שלא רואה את ה-images של Docker במחשב) הוא מנסה להוריד אותו מ-registry באינטרנט. ה-image קיים רק אצלנו, ולכן ההורדה נכשלת (`ErrImagePull`). `Never` אומר: לא להוריד, להשתמש רק במה שנמצא על המחשב.

**2. פענוח `POSTGRES_PASSWORD`.**
הפקודה: `echo c2hvcHBhc3M= | base64 -d`, והתוצאה `shoppass`. כלומר base64 הוא קידוד ולא הצפנה: כל מי שיכולה לקרוא את ה-Secret מפענחת אותו בשנייה. ההגנה האמיתית היא הרשאות (RBAC) על מי רשאית לקרוא Secrets, הצפנה ב-etcd, ולא לעשות commit לסיסמה אמיתית ל-Git.

**3. למה המוצרים שורדים מחיקת pod?**
הקובץ `postgres-pvc.yml` (ה-PersistentVolumeClaim), שמחובר ל-`/var/lib/postgresql/data` ב-`postgres-deployment.yml`. ה-pod נמחק, ה-PVC נשאר, וה-pod החדש מתחבר לאותה ארונית. מה שכן היה מוחק את הנתונים: `kubectl delete pvc postgres-pvc -n shoplist`.

**4. מה קורה אם ה-backend עולה לפני postgres?**
ה-backend לא מצליח להתחבר למסד הנתונים, ה-readinessProbe שלו נכשל והוא מסומן `0/1`. Kubernetes ממשיך לנסות עד שהכל מוכן, ואין צורך בסדר עלייה. מי שדואג שלא יגיעו אליו בקשות לפני שהוא מוכן הוא ה-readinessProbe: כל עוד הוא נכשל, ה-pod לא ברשימת ה-endpoints של ה-Service, וה-Service לא שולח אליו שיחות.

**5. ד8: מדבקה שגויה (`app: backnd`).**
*הערה: הסעיף הזה מתאר את ההתנהגות הצפויה. את ההרצה עצמה והצילום צריך לעשות ב-cluster, והתשובה כאן נכתבה בלי להריץ.*
בדפדפן: הנוריות של ה-API הופכות לאדומות והמוצרים לא נטענים, כי ה-frontend לא מקבל תשובה מ-`backend`. ה-pods של ה-backend **לא נפלו**: הם נשארו `1/1 Running`, כי ה-Deployment עדיין מוצא אותם לפי המדבקה הנכונה שלו. `kubectl get endpoints backend` מציג `<none>` (ריק), כי ה-Service מחפש pods עם `app: backnd` ואין כאלה. ה-Service נוצר בלי שום שגיאה, אבל אין מאחוריו אף pod. אחרי התיקון ל-`app: backend` וה-apply, הכתובות חוזרות ל-endpoints והנוריות ירוקות.

**6. שיניתי ערך ב-`app-config`, עשיתי `apply`, וה-backend עדיין רואה את הישן.**
משתני סביבה נקראים רק ברגע שה-pod עולה, ולכן pod שכבר רץ לא מתעדכן. הפתרון: `kubectl rollout restart deployment/backend -n shoplist`, שמקים pods חדשים שקוראים את הערך החדש.

**7. כמה פעמים `shoplist`?**
13 פעמים: פעם אחת בכל אחד מ-13 הקבצים (`namespace: shoplist`, ובקובץ `namespace.yml` עצמו `name: shoplist`). כדי להריץ עותק שני ב-`shoplist-test` צריך לשנות את כל 13 הקבצים. זה מה ש-Helm פותר בשיעור הבא: ערך אחד שחוזר, כתוב פעם אחת בלבד.

---

## בונוס ד12 — livenessProbe

**backend נתקע (רץ אבל לא עונה): מה קורה עם readiness בלבד, ומה עם liveness?**
עם readiness בלבד: ה-pod יוצא מה-endpoints והבקשות לא מגיעות אליו, אבל הוא **לא מופעל מחדש** ונשאר תקוע לתמיד. עם liveness: אחרי `failureThreshold` כישלונות ברצף (3), ה-kubelet מפעילה מחדש את ה-container, והוא חוזר לעבוד.
**למה liveness לא בודק את postgres?** אם postgres נופל לדקה, כל ה-backends ייכשלו ב-liveness ויופעלו מחדש שוב ושוב, בדיוק כשהמערכת חלשה. liveness בודק רק שהתהליך עצמו חי. readiness הוא זה שיכול לבדוק תלויות.
