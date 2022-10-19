
Pgcrypto설치
```sql
create extension pgcrypto;
create extension if not exists pgcrypto;
```

Schema 적용
> 위에 방식으로 설치하면 기본 schema에만 해당 외장 함수들이 적용 됨. 따라서 아래와 같이 각 스키마에 적용 시켜줘야 함.
 ```sql
 alter extension pgcrypto set schema schema이름
```

암호화
```sql
--암호화일 경우
encode(encrypt({해당컬럼명}::bytea, {암호화키}, 'AES'), 'base64')
--이렇게 형변환 할경우 특수문자가 포함되면 오류 발생
--따라서 이렇게 convert_to함수 사용하여 utf8로 인코딩 작업후 진행
encode(encrypt(convert_to({해당컬럼명}, 'UTF8'), {암호화키}, 'AES'), 'base64')
```
복호화
```sql
--복호화일 경우
convert_from(decrypt(decode({해당컬럼명}, 'base64'), {암호화키}, 'AES'),'utf8')
```

암복호화 예제
```sql
-- 테스트 데이터 만들기
create table tmp_key (
    key varchar(10),
    name varchar(100)
);
 
insert into tmp_key
values
('10', '가나다'),
('20', 'abc'),
('30', '123'),
('40', '가12a'),
('50', 'ekfff');
 
create table tmp_dat (
    key varchar(200),
    mn  integer
);
 
insert into tmp_dat
values
('10', 100),
('10', 200),
('10', 300),
('20', 1000),
('30', 2000),
('30', 50),
('40', 11),
('40', 21),
('40', 31),
('40', 41),
('50', 1);
 
--대량의 데이터 만들기 (여러번 실행하여)
insert into tmp_dat
select key, mn from tmp_dat;
 
--일반조인
select tmp_dat.*, name from tmp_dat
left join tmp_key on tmp_dat.key = tmp_key.key
 
--복호화 후 조인
select tmp_dat.*, name
from tmp_dat left join tmp_key on convert_from(decrypt(decode(tmp_dat.key, 'base64'), '7a11b8b3f1e48b771808bd437a29181b', 'AES'), 'utf8') = tmp_key.key
 
-- 복호화
update tmp_dat
set key = convert_from(decrypt(decode(tmp_dat.key, 'base64'), '7a11b8b3f1e48b771808bd437a29181b', 'AES'), 'utf8');
 
--암호화
update tmp_dat
set key = encode(encrypt(tmp_dat.key::bytea, '7a11b8b3f1e48b771808bd437a29181b', 'AES'), 'base64');
```
> 이방식으로 join시에도 이상없이 join되는 부분 확인. 

> python script로 암복호화 내용 동일한 부분 확인

이슈사항
> 특정케이스인지 몰라도 테이블을 join할 경우 on절에 복호화로직 적용할 경우 속도가 느려지는 상황 발생(기존 0.5초 -> 14.3초)

> Temp table 생성하는 방향으로 수정
```sql
--방법1
  --기존 table을 임시 테이블로 만들고
  create temp table tmp_table on commit drop as
  select * from table;

  --임시 테이블의 값을 복호화 한후에
  update tmp_table
  set 복호화컬럼1 = convert_from(decrypt(decode({복호화컬럼1}, 'base64'), {암호화키}, 'AES'),'utf8'),
  복호화컬럼2 = convert_from(decrypt(decode({복호화컬럼2}, 'base64'), {암호화키}, 'AES'),'utf8');
  
  --해당 임시 table을 기존 테이블 대신 사용
  
  
--방법2
  --임시 테이블 생성시 암호화 컬럼만 복호화 하여 생성
  create temp table tmp_table on commit drop as
  select 해당 table pl,
  convert_from(decrypt(decode({복호화컬럼1}, 'base64'), {암호화키}, 'AES'),'utf8') as {복호화컬럼1},
  convert_from(decrypt(decode({복호화컬럼2}, 'base64'), {암호화키}, 'AES'),'utf8') as {복호화컬럼2}
  from table;
  
  --해당 임시 table을 기존 테이블과 join하여 사용
```
