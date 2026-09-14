---
name: typescript
description: TypeScriptで実装する際のエラーハンドリング規約。エラーの表現には@praha/error-factoryを利用したカスタムエラーを利用し、エラー発生時は原則throwせず@praha/byethrowのResult型で返す。「TypeScriptで実装して」「エラーハンドリングして」のようにTypeScriptの実装を行う時に使う。
---
# typescript

TypeScript の実装では、以下のエラーハンドリング規約に従う。

## 1. エラーの表現には @praha/error-factory を利用する

- 素の `Error` や文字列リテラルを直接 `Result.fail()` に渡さない。`ErrorFactory` から生成したカスタムエラークラスを利用する。
- `name` は明示的に指定し、`error.name` での判別・`instanceof` での絞り込みができるようにする。
- 追加情報が必要な場合は `fields` を利用し、動的メッセージが必要な場合は関数形式の `message` を利用する。
- 元のエラーは `cause` に保持する。

```ts
import { ErrorFactory } from '@praha/error-factory';

class ValidationError extends ErrorFactory({
  name: 'ValidationError',
  message: 'Invalid input provided',
}) {}

class QueryError extends ErrorFactory({
  name: 'QueryError',
  message: 'An error occurred while executing a query',
  fields: ErrorFactory.fields<{ query: string }>(),
}) {}

class NotFoundError extends ErrorFactory({
  name: 'NotFoundError',
  message: 'Resource not found',
}) {}

// cause の保持
const error = new QueryError({ query: 'SELECT * FROM users', cause: unknownError });
```

## 2. エラー発生時は原則 throw しない。@praha/byethrow の Result 型を返す

- 失敗しうる関数は例外を `throw` せず、`Result<T, E>` / `ResultAsync<T, E>` を返す。
- `throw` / `try-catch` をビジネスロジック内に書かない。例外を投げうる処理は `Result.try` または `Result.fn` でラップし、カスタムエラーに変換して `Failure` として返す。
- 呼び出し側は `Result.isSuccess` / `Result.isFailure` で判定するか、`Result.pipe` + `andThen` / `map` / `mapError` / `orElse` で合成する。
- 予期しないインフラエラー（DB接続失敗、ネットワーク障害など）のみ `throw` を許容する。その場合も可能な限り `Result.fn` で `UnexpectedError` 等のカスタムエラーに包み、スタックトレースと `cause` を残す。

```ts
import { Result } from '@praha/byethrow';

// 失敗しうる関数は Result を返す
const validateId = (id: string): Result.Result<string, ValidationError> => {
  if (!id.startsWith('u')) {
    return Result.fail(new ValidationError());
  }
  return Result.succeed(id);
};

const executeQuery = (sql: string): Result.ResultAsync<QueryResult, QueryError> => {
  return Result.try({
    try: () => database.query(sql),
    catch: (error) => new QueryError({ query: sql, cause: error }),
  });
};

// 合成
const findUser = (id: string) => {
  return Result.pipe(
    validateId(id),
    Result.andThen((validId) => executeQuery(`SELECT * FROM users WHERE id = '${validId}'`)),
    Result.andThen((row) => {
      if (!row) {
        return Result.fail(new NotFoundError());
      }
      return Result.succeed({ id: row.id, name: row.name });
    }),
  );
};

// 呼び出し側
const result = await findUser('u123');
if (Result.isSuccess(result)) {
  console.log(result.value);
} else {
  switch (result.error.name) {
    case 'ValidationError':
    case 'QueryError':
    case 'NotFoundError':
      console.error(result.error.message);
      break;
  }
}
```

## 3. 詳細は公式ドキュメントを参照する

- `npx @praha/byethrow-docs list --query "<keyword>"` や `npx @praha/byethrow-docs search "<query>"` で API の詳細・用例を確認する。
- 不明点は https://praha-inc.github.io/byethrow/ および https://github.com/praha-inc/error-factory を参照する。
