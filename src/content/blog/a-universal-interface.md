---
title: "A Universal Interface?"
description: "Are universal interfaces always the best option?"
pubDate: "Nov 13 2023"
heroImage: "/a-universal-interface.jpg"
---

Throughout my learning journey, the emphasis on utilizing interfaces for **processes** reliant on external dependencies has been steadfast. This practice offers the benefit that in the event of changing to a different dependency, the consumers of the interface won't need to refactor.

However, I've discerned that there are instances where defining a universal interface accommodating all potential implementations proves challenging. This is especially true when lacking control over the definition of the request parameters for executing a **process**.

Before delving into these instances, let's take a look at an example where a universal interface can be defined. Imagine we're creating a social network platform, and a user intends to add another as a friend. This action necessitates fetching both users from a database. By employing an interface designed for database operations, any future shifts in our choice of databases won't require consumers of these interfaces to undertake extensive refactoring. Let's examine this example in detail:

```ts
// TypeScript example
class User {
  id: string;
  // other fields...

  constructor(id: string) {
    this.id = id;
  }
}

class AddFriendUseCase {
  userRespository: UserRepository;

  constructor(userRespository: UserRepository) {
    this.userRespository = userRespository;
  }

  async execute(requestUserId: string, otherUserId: string) {
    const requestUser = await this.userRespository.findById(requestUserId);
    // ...
  }
}

interface UserRepository {
  findById: (id: string) => Promise<User | undefined>;
}

class UserRepositoryPostgresql implements UserRepository {
  pool: Pool;

  constructor(pool: Pool) {
    this.pool = pool;
  }

  async findById(id: string): Promise<User | undefined> {
    const { rows } = await this.pool.query("select * from user where id = $1", [
      id,
    ]);
    if (rows.length === 0) return undefined;
    return new User(rows[0].id);
  }
}

class UserRepositoryMongodb implements UserRepository {
  client: MongoClient;

  constructor(client: MongoClient) {
    this.client = client;
  }

  async findById(id: string): Promise<User | undefined> {
    const userDocument = await this.client
      .db("db")
      .collection("user")
      .findOne({ id });
    return userDocument == null ? undefined : new User(userDocument.id);
  }
}
```

As demonstrated in this example, if we need to switch databases, the consumers (AddFriendUseCase) of the interface (UserRepository) don't need to alter their code. This is possible because the implementations (UserRepositoryPostgresql and UserRepositoryMongodb) adhere to a common contract. In this case, the query languages of PostgreSQL and MongoDB allow us to retrieve data using a wide range of parameters and those parameters are defined by us, thus enabling the creation of a universal interface.

However, when dealing with, for example, a SaaS third-party API, the parameters for executing a process or query are set by this third party, not us. This means that there are situations where we won't be able to interact with this API in the same way we would with our database. Consequently, creating a universal interface might not be feasible.

To illustrate, let's consider a practical scenario. Suppose we offer our users the capability to purchase cryptocurrencies, a task that necessitates engaging a third-party cryptocurrency service provider. For the purposes of this example, I'll demonstrate using the Fortress Trust APIs, accessible [here](https://developers.fortress.io/reference/post_api-trust-v1-trades-1). If we want to establish a universal interface compatible with various cryptocurrency providers, we could structure it as follows:

```ts
interface CryptoCurrencyProvider {
  buyCrypto(userId: string, usdAmount: number, symbol: string): Promise<void>;
}

class FortressCryptoCurrencyProvider implements CryptoCurrencyProvider {
  constructor() {}

  async buyCrypto(
    userId: string,
    usdAmount: number,
    symbol: string
  ): Promise<void> {
    const body = {
      // accountId: ???,
      type: "market",
      from: {
        asset: "usd",
        amount: usdAmount,
      },
      to: {
        asset: symbol,
        network: this.currencyNetwork(symbol),
      },
    };
    const url = "https://example.com/api/trust/v1/trades";
    await fetch(url, {
      method: "POST",
      // headers: { 'Authorization': `Bearer ${???}` },
      body: JSON.stringify(body),
    });
  }

  currencyNetwork(symbol: string): string {
    // mock response
    return "";
  }
}
```

As we get down to coding, we realize that Fortress demands two things: an `accountId` and an expiring Json Web Token. These are exclusive to Fortress APIs, throwing a wrench into our plan for a universal interface. Unless, of course, we fetch these parameters right within the `buyCrypto` method. Let's give that a shot:

```ts
interface Database {
  findFortressAccountId(userId: string): Promise<string | undefined>;
  getCachedFortressJwt(): Promise<string | undefined>;
  updateCachedFortressJwt(val: string): Promise<void>;
}

class FortressCryptoCurrencyProvider implements CryptoCurrencyProvider {
  db: Database;
  // config secrets
  username: string;
  password: string;
  clientId: string;

  constructor(db: Database) {
    this.db = db;
    this.username = process.env.FORTRESS_USERNAME!;
    this.password = process.env.FORTRESS_PASSWORD!;
    this.clientId = process.env.FORTRESS_CLIENTID!;
  }

  async buyCrypto(
    userId: string,
    usdAmount: number,
    symbol: string
  ): Promise<void> {
    const jwt = await this.fetchJwt();
    const accountId = await this.db.findFortressAccountId(userId);
    if (accountId === undefined) throw new Error();

    const body = {
      accountId,
      type: "market",
      from: {
        asset: "usd",
        amount: usdAmount,
      },
      to: {
        asset: symbol,
        network: this.currencyNetwork(symbol),
      },
    };
    const url = "https://example.com/api/trust/v1/trades";
    await fetch(url, {
      method: "POST",
      headers: { Authorization: `Bearer ${jwt}` },
      body: JSON.stringify(body),
    });
  }

  currencyNetwork(symbol: string): string {
    // mock response
    return "";
  }

  async fetchJwt(): Promise<string> {
    const cachedJwt = await this.db.getCachedFortressJwt();
    if (
      cachedJwt === undefined ||
      this.parseJwt(cachedJwt).expiresAt <= new Date()
    ) {
      const jwt = await this.fetchNewJwt();
      await this.db.updateCachedFortressJwt(jwt);
      return jwt;
    }
    return cachedJwt;
  }

  async fetchNewJwt(): Promise<string> {
    const body = {
      username: this.username,
      password: this.password,
      clientId: this.clientId,
    };
    const url = "https://example.com/oauth/token";
    const res = await fetch(url, {
      method: "POST",
      body: JSON.stringify(body),
    });
    const jsonRes = await res.json();
    return jsonRes.AccessToken;
  }

  parseJwt(jwt: string): FortressJwtPayload {
    // mock response
    return { jwt, expiresAt: new Date() };
  }
}

type FortressJwtPayload = {
  jwt: string;
  expiresAt: Date;
};
```

This looks like a pretty good solution, we encapsulate the implementation details thus we can reuse our interface with other implementations. However, there is a tradeoff, whenever we use this method, it triggers two calls to our database that could be omitted - one for the user fortress account id and another for the cached JWT token. This could potentially put more strain on our system. Consider a scenario where a new feature requires us to run a scheduled cron job that purchases $20 worth of each of the top 10 cryptocurrencies for all users.

```ts
class ScheduledCryptoPurchasesJob {
  cryptoCurrencyProvider: CryptoCurrencyProvider;

  constructor(cryptoCurrencyProvider: CryptoCurrencyProvider) {
    this.cryptoCurrencyProvider = cryptoCurrencyProvider;
  }

  async execute(userIds: string[], topTenCryptos: string[]) {
    for (const userId of userIds) {
      for (const crypto of topTenCryptos) {
        await this.cryptoCurrencyProvider.buyCrypto(userId, 20, crypto);
      }
    }
  }
}
```

All of a sudden, instead of making just one call for the cached JWT token at the start, we're now fetching it ten times for each user. And instead of grabbing the fortress account ID once per user, we're doing it ten times. So, if we've got 1000 users, that's a whopping 20,000 calls! But we could streamline it down to just 1001 calls. That's a massive 94.995% drop in database calls. Imagine if each database call takes about 180 milliseconds – right now, it'd take a whole hour to finish all those. But with the more efficient approach, we'd wrap it up in just 3.336 minutes.

While avoiding premature optimization is important, we shouldn't disregard performance entirely. The critical area for optimization is in managing I/O operations. Unlike the speedy nature of in-memory operations, network calls—particularly those traveling over the internet—often bring about noticeable delays. This leads us to the question: is it justifiable to prioritize maintainability by concealing implementation details at the potential expense of additional unnecessary I/O operations?

Let's see what our code could look like in the alternative implementation:

```ts
interface CryptoCurrencyProvider {
  fortress: FortressApi;
}

interface FortressApi {
  buyCrypto(
    jwt: string,
    accountId: string,
    usdAmount: number,
    symbol: string
  ): Promise<void>;
  fetchJwt(cachedJwt?: string): Promise<string>;
  findAccountId(userId: string): Promise<string | undefined>;
}

class ScheduledCryptoPurchasesJob {
  cryptoCurrencyProvider: CryptoCurrencyProvider;

  constructor(cryptoCurrencyProvider: CryptoCurrencyProvider) {
    this.cryptoCurrencyProvider = cryptoCurrencyProvider;
  }

  async execute(userIds: string[], topTenCryptos: string[]) {
    const jwt = await this.cryptoCurrencyProvider.fortress.fetchJwt();
    for (const userId of userIds) {
      const accountId =
        await this.cryptoCurrencyProvider.fortress.findAccountId(userId);
      if (accountId === undefined) continue;

      for (const crypto of topTenCryptos) {
        await this.cryptoCurrencyProvider.fortress.buyCrypto(
          jwt,
          accountId,
          10,
          crypto
        );
      }
    }
  }
}

class CryptoCurrencyProviderImpl implements CryptoCurrencyProvider {
  fortress: FortressApi;

  constructor(fortress: FortressApi) {
    this.fortress = fortress;
  }
}

interface Database {
  findFortressAccountId(userId: string): Promise<string | undefined>;
  getCachedFortressJwt(): Promise<string | undefined>;
  updateCachedFortressJwt(val: string): Promise<void>;
}

class FortressApiImpl implements FortressApi {
  db: Database;
  // config secrets
  username: string;
  password: string;
  clientId: string;

  constructor(db: Database) {
    this.db = db;
    this.username = process.env.FORTRESS_USERNAME!;
    this.password = process.env.FORTRESS_PASSWORD!;
    this.clientId = process.env.FORTRESS_CLIENTID!;
  }

  async findAccountId(userId: string): Promise<string | undefined> {
    return this.db.findFortressAccountId(userId);
  }

  async buyCrypto(
    jwt: string,
    accountId: string,
    usdAmount: number,
    symbol: string
  ): Promise<void> {
    return this.trade(
      accountId,
      "market",
      {
        amount: usdAmount,
        asset: "usd",
      },
      {
        asset: symbol,
      },
      jwt
    );
  }

  private async trade(
    accountId: string,
    orderType: string,
    from: {
      asset: string;
      amount: number;
    },
    to: {
      asset: string;
    },
    jwt: string
  ): Promise<void> {
    const body = {
      accountId,
      type: orderType,
      from: {
        asset: from.asset,
        network: this.currencyNetwork(from.asset),
        amount: from.amount,
      },
      to: {
        asset: to.asset,
        network: this.currencyNetwork(to.asset),
      },
    };
    const url = "https://example.com/api/trust/v1/trades";
    await fetch(url, {
      method: "POST",
      headers: { Authorization: `Bearer ${jwt}` },
      body: JSON.stringify(body),
    });
  }

  async fetchJwt(): Promise<string> {
    const cachedJwt = await this.db.getCachedFortressJwt();
    if (
      cachedJwt === undefined ||
      this.parseJwt(cachedJwt).expiresAt <= new Date()
    ) {
      const jwt = await this.fetchNewJwt();
      await this.db.updateCachedFortressJwt(jwt);
      return jwt;
    } else return cachedJwt;
  }

  private async fetchNewJwt(): Promise<string> {
    const body = {
      username: this.username,
      password: this.password,
      clientId: this.clientId,
    };
    const url = "https://example.com/oauth/token";
    const res = await fetch(url, {
      method: "POST",
      body: JSON.stringify(body),
    });
    const jsonRes = await res.json();
    return jsonRes.AccessToken;
  }

  private currencyNetwork(symbol: string): string {
    // mock response
    return "";
  }

  private parseJwt(jwt: string): FortressJwtPayload {
    // mock response
    return { jwt, expiresAt: new Date() };
  }
}

type FortressJwtPayload = {
  jwt: string;
  expiresAt: Date;
};
```

If we want to add a new crypto currency provider we would create a new interface for it and add it as a field to the `CryptoCurrencyProvider` interface. This is an important trade off to consider, because now everytime we want to change implementations, the consumers of the interface will have to change too, each of them will have to decide which implementation to use. I personally consider this trade off worth it, I don't like the idea of trying to force an unforceable universal interface or the idea of making unnecessary I/O operations. However, this choice still troubles me a bit, I am open to different perspectives on the matter and would greatly appreciate hearing alternative opinions from anyone out there.