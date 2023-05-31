---
title: '@nestjs/bull に依存したクラスのテスト'
emoji: '🐈‍⬛'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['NestJS', 'Bull']
published: true
---

# 始めに

最近 [@nestjs/bull](https://github.com/nestjs/bull) を使った非同期ジョブの実装をしていて、テストを書くのに少し手間どったので解決策の共有になります。
ユニットテストと E2E テストでモックの内容が異なるのでそれぞれ記載しています。

また、テストコードは[公式サンプル](https://github.com/nestjs/nest/tree/master/sample/26-queues)を元に作成しています。

# ユニットテストの場合

`getQueueToken()` をモックすることでテストできます。

```ts:audio.controller.spec.ts
import { getQueueToken } from '@nestjs/bull';
import { Test, TestingModule } from '@nestjs/testing';
import { AudioController } from './audio.controller';

describe('AudioController', () => {
  let audioController: AudioController;

  const mockQueue = {
    add: jest.fn().mockName('AudioQueue.add')
  }

  beforeEach(async () => {
    const moduleRef: TestingModule = await Test.createTestingModule({
      controllers: [AudioController],
      providers: [
        {
          provide: getQueueToken('audio'),
          useValue: mockQueue,
        }
      ],
    }).compile();

    audioController = moduleRef.get(AudioController);
  });

  describe('transcode', () => {
    it('should queue added', async () => {
      await audioController.transcode();

      expect(mockQueue.add).toBeCalled();
    });
  });
});
```

# E2E テストの場合

E2E のようにモジュール単位でテストする場合は `overrideProvider()` を使うことでモックできます。
注意点として、モジュール内に Consumer が含まれる場合は定義しているメソッドによって Queue に対して追加でメソッドのモックが必要です（おそらく、Queue を遅延実行していない場合は即座に process が実行されてしまうため）

```ts:audio.e2e.spec.ts
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import { AudioModule } from './audio.module';
import { getQueueToken } from '@nestjs/bull';

describe('e2e - Audio', () => {
  let app: INestApplication;

  const mockQueue = {
    add: jest.fn().mockName('AudioQueue.add'),
    // @Process() を定義している場合は必要
    process: jest.fn().mockName('AudioQueue.process'),
    /**
     * @OnXXX() を定義している場合は必要
     * @see https://docs.nestjs.com/techniques/queues#event-listeners
     */
    on: jest.fn().mockName('AudioQueue.add'),
  };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AudioModule],
    })
      .overrideProvider(getQueueToken('audio'))
      .useValue(mockQueue)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/POST transcode`, async () => {
    await request(app.getHttpServer())
      .post('/audio/transcode')
      .expect(201);

      expect(mockQueue.add).toBeCalled();
  });

  afterAll(async () => {
    await app.close();
  });
});
```

# 参考

- [typescript - How can a Nest Bull queue be tested via Jest (DI via @InjectQueue)? - Stack Overflow](https://stackoverflow.com/questions/68214492/how-can-a-nest-bull-queue-be-tested-via-jest-di-via-injectqueue)
- [testing - Mocking Bull queues in NestJS - Stack Overflow](https://stackoverflow.com/questions/70563784/mocking-bull-queues-in-nestjs)
- [Testing | NestJS - A progressive Node.js framework](https://docs.nestjs.com/fundamentals/testing)
