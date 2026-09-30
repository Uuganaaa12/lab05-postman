# AI-ийн гаргасан тест дизайн

Claude-д server.js-ийг өгөөд "Энэ API-гийн POST /registrations функцийн сонголтуудыг тодорхойлж, сонголт бүрийг эквивалент ангид хуваагаад, ангиудын хослолоор Postman тестүүд үүсгэ" гэж асуусан хариу. Засаагүй.

---

## 1. Функц ба түүний сонголтууд

Бид `POST /registrations` функцийг тестлэнэ. Оролт нь body доторх `studentID` ба `courseID`. Гэхдээ хариу нь серверийн төлөвөөс ч хамаардаг: оюутны status, coursesTaken, хичээлийн prerequisites болон өмнө хийгдсэн бүртгэлүүд. Тиймээс эдгээрийг ч бас сонголт гэж тооцно.

1. Хүсэлтийн биеийн формат
2. `studentID`
3. `courseID`
4. Оюутны `status`
5. Хичээлийн өмнөх нөхцлийн (prerequisites) тоо
6. Оюутны өгсөн хичээлүүд өмнөх нөхцлийг хангаж байгаа эсэх
7. Оюутан тухайн хичээлд өмнө нь бүртгүүлсэн эсэх

## 2. Эквивалент ангиуд

| Сонголт | Анги | Жишээ утга | Хүлээгдэх хариу |
|---|---|---|---|
| 1. Биеийн формат | B1 Зөв JSON объект | `{"studentID":"s1","courseID":"CS101"}` | дараагийн шалгалт руу шилжинэ |
| | B2 Буруу JSON | `{"studentID":"s1",` | 400 `ERROR_BAD_JSON` |
| | B3 Хоосон объект | `{}` | 400 `ERROR_BAD_REQUEST` |
| 2. studentID | S1 Бүртгэлтэй | `s1` | – |
| | S2 Бүртгэлгүй | `s999` | 200 `ERROR_NO_STUDENT` |
| | S3 Талбар байхгүй эсвэл хоосон | талбаргүй, `""` | 400 `ERROR_BAD_REQUEST` |
| 3. courseID | K1 Бүртгэлтэй | `CS101` | – |
| | K2 Бүртгэлгүй | `CS999` | 200 `ERROR_NO_COURSE` |
| | K3 Талбар байхгүй эсвэл хоосон | талбаргүй, `""` | 400 `ERROR_BAD_REQUEST` |
| 4. status | T1 `"active"` | `active` | – |
| | T2 Бусад утга | `inactive` | 200 `ERROR_INACTIVE_STUDENT` |
| 5. Prerequisites тоо | P0 Байхгүй | `[]` | – |
| | P1 Нэг | `["CS101"]` | – |
| | P2 Хоёр ба түүнээс олон | `["CS101","CS102"]` | – |
| 6. Өмнөх нөхцөл хангасан эсэх | H1 Бүгдийг өгсөн | | 201 `OK` |
| | H2 Заримыг нь өгсөн | | 200 `ERROR_PREREQUISITES`, `missing` = дутуу хичээлүүд |
| | H3 Нэгийг ч өгөөгүй | | 200 `ERROR_PREREQUISITES`, `missing` = бүгд |
| 7. Өмнө нь бүртгүүлсэн | D1 Үгүй | | 201 |
| | D2 Тийм | | Код үүнийг шалгадаггүй, 201 буцаана |

Үүсэх боломжгүй хослолууд: P0 үед H2, H3 байж болохгүй. P1 үед H2 байж болохгүй, учир нь хичээл ганцхан өмнөх нөхцөлтэй бол "заримыг нь өгсөн" гэдэг нь "нэгийг ч өгөөгүй"-тэй адилхан. S2, S3, K2, K3, T2 тохиолдолд функц эрт буцдаг тул P, H, D-ийн утга ач холбогдолгүй.

## 3. Хослуулах арга

- S1, K1, T1, D1 гэсэн зөв ангиудыг тогтмол барина. Үүн дээр P ба H-ийн боломжит бүх хослолыг шалгана: P0-H1, P1-H1, P1-H3, P2-H1, P2-H2, P2-H3.
- Буруу анги бүрийг ганцаар нь шалгана, бусад нь зөв байна (нэг тестэд нэг алдаа).
- Хоёр алдаа зэрэг гарвал аль нь эхэлж шалгагдахыг 2 тестээр шалгана.

## 4. Тестийн өгөгдөл (Initialization)

Environment-д `baseUrl = http://localhost:3000` гэж тохируулна. Дараа нь "00 Setup" хавтсанд эдгээр хүсэлтүүдийг үүсгэнэ (Body нь raw, JSON төрөлтэй):

| Хүсэлт | Body |
|---|---|
| `PUT {{baseUrl}}/students/s1` | `{"status":"active","coursesTaken":["CS101","CS102"]}` |
| `PUT {{baseUrl}}/students/s2` | `{"status":"active","coursesTaken":["CS101"]}` |
| `PUT {{baseUrl}}/students/s3` | `{"status":"active","coursesTaken":[]}` |
| `PUT {{baseUrl}}/students/s4` | `{"status":"inactive","coursesTaken":["CS101","CS102"]}` |
| `PUT {{baseUrl}}/courses/CS101` | `{"prerequisites":[]}` |
| `PUT {{baseUrl}}/courses/CS201` | `{"prerequisites":["CS101"]}` |
| `PUT {{baseUrl}}/courses/CS301` | `{"prerequisites":["CS101","CS102"]}` |

Setup хүсэлт бүрийн Tests хэсэгт:
```js
pm.test("Setup OK", () => {
  pm.response.to.have.status(200);
  pm.expect(pm.response.json().result).to.eql("OK");
});
```

## 5. Тест кейсүүд

Бүх тест `POST {{baseUrl}}/registrations` рүү явна.

| TC | Ангиуд | Body | Хүлээгдэх |
|---|---|---|---|
| TC01 | S1 K1 T1 P0 D1 | `{"studentID":"s1","courseID":"CS101"}` | 201, `OK`, registrationID |
| TC02 | P1 H1 | `{"studentID":"s2","courseID":"CS201"}` | 201, `OK` |
| TC03 | P2 H1 | `{"studentID":"s1","courseID":"CS301"}` | 201, `OK` |
| TC04 | P0, coursesTaken хоосон | `{"studentID":"s3","courseID":"CS101"}` | 201, `OK` |
| TC05 | P1 H3 | `{"studentID":"s3","courseID":"CS201"}` | 200, `ERROR_PREREQUISITES`, missing `["CS101"]` |
| TC06 | P2 H2 | `{"studentID":"s2","courseID":"CS301"}` | 200, `ERROR_PREREQUISITES`, missing `["CS102"]` |
| TC07 | P2 H3 | `{"studentID":"s3","courseID":"CS301"}` | 200, `ERROR_PREREQUISITES`, missing `["CS101","CS102"]` |
| TC08 | S2 | `{"studentID":"s999","courseID":"CS101"}` | 200, `ERROR_NO_STUDENT` |
| TC09 | T2 | `{"studentID":"s4","courseID":"CS101"}` | 200, `ERROR_INACTIVE_STUDENT` |
| TC10 | K2 | `{"studentID":"s1","courseID":"CS999"}` | 200, `ERROR_NO_COURSE` |
| TC11 | S3 (талбаргүй) | `{"courseID":"CS101"}` | 400, `ERROR_BAD_REQUEST` |
| TC12 | K3 (хоосон) | `{"studentID":"s1","courseID":""}` | 400, `ERROR_BAD_REQUEST` |
| TC13 | B3 | `{}` | 400, `ERROR_BAD_REQUEST` |
| TC14 | B2 | `{"studentID":"s1",` | 400, `ERROR_BAD_JSON` |
| TC15 | S2 + K2 (дараалал) | `{"studentID":"s999","courseID":"CS999"}` | 200, `ERROR_NO_STUDENT` |
| TC16 | T2 + K2 (дараалал) | `{"studentID":"s4","courseID":"CS999"}` | 200, `ERROR_INACTIVE_STUDENT` |
| TC17 | `GET /registrations` | – | Зөвхөн TC01–TC04-ийн 4 бүртгэл байх ёстой |
| TC18 | D2 | `{"studentID":"s1","courseID":"CS101"}` | Код 201 буцаана (доорх тайлбарыг хар) |

TC18-ын тайлбар: код давхар бүртгэлийг шалгадаггүй. Лекцийн спецэд давхар бүртгэлийг хориглосон бол энд 200 + `ERROR_*` хүлээж тест бич. Тест унавал түүнийг согог гэж тэмдэглэнэ.

## 6. Postman Tests скриптүүд

Амжилттай бүртгэл (TC01–TC04):
```js
pm.test("201 Created", () => pm.response.to.have.status(201));
pm.test("result OK, registrationID байна", () => {
  const j = pm.response.json();
  pm.expect(j.result).to.eql("OK");
  pm.expect(j.registrationID).to.be.a("number");
});
```

Өмнөх нөхцөлийн алдаа (TC05–TC07). `missing` утгыг кейс бүрт тохируулна:
```js
pm.test("200", () => pm.response.to.have.status(200));
pm.test("ERROR_PREREQUISITES ба missing зөв", () => {
  const j = pm.response.json();
  pm.expect(j.result).to.eql("ERROR_PREREQUISITES");
  pm.expect(j.missing).to.eql(["CS101", "CS102"]);
  pm.expect(j).to.not.have.property("registrationID");
});
```

200 + ERROR (TC08–TC10, TC15, TC16). Алдааны нэрийг кейс бүрт солино:
```js
pm.test("200", () => pm.response.to.have.status(200));
pm.test("ERROR_NO_STUDENT", () => {
  const j = pm.response.json();
  pm.expect(j.result).to.eql("ERROR_NO_STUDENT");
  pm.expect(j).to.not.have.property("registrationID");
});
```

400 (TC11–TC14). TC14 дээр `ERROR_BAD_JSON` гэж солино:
```js
pm.test("400", () => pm.response.to.have.status(400));
pm.test("ERROR_BAD_REQUEST", () =>
  pm.expect(pm.response.json().result).to.eql("ERROR_BAD_REQUEST"));
```

TC17 `GET {{baseUrl}}/registrations`:
```js
pm.test("Алдаатай хүсэлтүүд бүртгэл үүсгээгүй", () => {
  const list = pm.response.json();
  pm.expect(list).to.have.lengthOf(4);
  pm.expect(list.map(r => r.registrationID)).to.eql([1, 2, 3, 4]);
});
```

## 7. Кодоос олдсон согогууд

Эдгээр тестүүдийг хамгийн сүүлд, тусад нь ажиллуул. Ялангуяа TC20, TC21 серверийг унагаадаг.

| TC | Body | Хүлээгдэх | Бодит үр дүн |
|---|---|---|---|
| TC19 | `{"studentID":"toString","courseID":"CS101"}` | 200 `ERROR_NO_STUDENT` | 200 `ERROR_INACTIVE_STUDENT` |
| TC20 | `null` | 400 `ERROR_BAD_REQUEST` | Сервер унана (Postman дээр "socket hang up" гарна) |
| TC21 | `{"studentID":"s1","courseID":"constructor"}` | 200 `ERROR_NO_COURSE` | Сервер унана |

Шалтгаанууд:
- `students`, `courses` нь энгийн `{}` объект. Тиймээс `toString`, `constructor` гэх мэт `Object.prototype`-оос удамшсан нэрс "олдчихдог". TC21 дээр `course.prerequisites` нь `undefined` тул `.filter` дуудахад TypeError гарна. Засах арга: `Object.hasOwn(students, studentID)` ашиглах, эсвэл `Map` руу шилжих.
- Body `null` үед `JSON.parse` нь `null` буцаана. Дараа нь `const { studentID, courseID } = data` мөр TypeError шиднэ. Засах арга: `if (!data || typeof data !== 'object')` гэсэн шалгалт нэмэх.

## Ажиллуулах

Өгөгдөл санах ойд хадгалагддаг. Тиймээс ажиллуулах бүрийн өмнө серверийг дахин асаана (`node server.js`). Дараа нь Collection Runner-ээр "00 Setup", TC01–TC18 дарааллаар ажиллуулна. Ингэхгүй бол registrationID болон TC17-ийн тоо таарахгүй. TC20 серверийг унагаах тул TC21-ийн өмнө серверийг дахин асааж, Setup-ыг дахин ажиллуул.
