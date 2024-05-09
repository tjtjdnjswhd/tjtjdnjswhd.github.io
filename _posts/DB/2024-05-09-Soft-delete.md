---
layout: post
title: Soft delete
date: 2024-05-09 18:00 +0900
categories: [DB]
tags: [softdelete, db]

---

## 설명

Soft delete란 DB에서 데이터를 물리적으로 삭제하는 대신 column의 값을 변경해 논리적으로 삭제된 것으로 보는 것입니다.

테이블을 분할하는 방법은 이 포스팅에선 다루지 않습니다.

## 사용 상황

* 데이터를 보존해야 할 필요가 있을 때(개인정보보호 법 등)

* 데이터 분석을 통한 마케팅, 기획개발, 서비스 개선이 필요할 때

* 삭제 취소 기능이 필요할 때

## 장단점

* 장점
  
  * 물리적 삭제보다 삭제/복구가 빠르고 간단함

  * 히스토리 추적의 용이함

* 단점

  * 데이터 크기가 증가해 쿼리 시 성능 저하가 있을 수 있음

  * index, unique 등 제약조건이 필요할 시 해당 column에 대해 고려해야 함

## 구현

table에 삭제됬음을 표시할 IsDeleted column를 추가합니다.

```sql
/* 생성 */
CREATE TABLE [dbo].[User] 
(
  [Id] INT IDENTITY(1, 1) PRIMARY KEY,
  [Name] NVARCHAR(45) NOT NULL,
  [Email] NVARCHAR(255) NOT NULL,
  [IsDeleted] BIT NOT NULL
);

/* 수정 */
Alter TABLE [dbo].[User] ADD IsDeleted BIT NOT NULL
```

instead of delete trigger를 추가합니다.

```sql
CREATE TRIGGER dbo.TR_User_OnDelete 
ON [dbo].[User]
INSTEAD OF DELETE
AS
BEGIN
SET NOCOUNT ON;
UPDATE [dbo].[User] SET [u].[IsDeleted] = CAST(1 AS BIT) WHERE [Id] IN (SELECT Id FROM DELETED);
END
```

필터링할 view을 추가합니다.

```sql
CREATE VIEW [dbo].[User_Filtered]
AS
SELECT [Id], [Name], [Email] FROM dbo.User WHERE [IsDeleted] = 0;
```

## index

구현 직후 대부분의 쿼리에는 삭제된 데이터가 적기 때문에 알아차리기 어렵지만 최악의 경우 모든 행의 IsDeleted를 확인해 필터링 합니다.

이를 방지하기 위해 인덱스를 추가해야 하는데, SQL Server의 경우 [Filtered indexes](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes?view=sql-server-ver16)를 지원합니다.

```sql
/* filtered indexes를 지원할 경우 */
CREATE INDEX [dbo].[IX_IsDeleted_filter] ON [dbo].[User] (IsDeleted) WHERE IsDeleted = 0;

/* filtered indexes를 지원하지 않는 경우 */
CREATE INDEX [dbo].[IX_IsDeleted] ON [dbo].[User] (IsDeleted)
```

## unique index 제약

table에 unique 제약사항을 추가할 때 IsDeleted를 포함한 복합 인덱스를 고려해야 합니다.

예를 들어 탈퇴한 유저가 재가입 할 경우, 유저가 같은 이메일을 쓸 수 있어야 합니다.

이를 위해 이메일과 IsDeleted를 포함한 복합 인덱스가 필요합니다.

```sql
CREATE OR ALTER UNIQUE INDEX [dbo].[IX_Email] ON [dbo].[User] (Email, IsDeleted);
```

## 기타

IsDeleted 대신 DeletedAt을 사용해 삭제된 시각을 저장하는 방법도 있습니다.
