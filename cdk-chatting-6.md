# CDK Chatting #6 2025/04/16

## 同じスタックを環境毎に使いまわしたい場合、bin/app.ts で静的/動的に複数作ったり分岐したりするよく見るパターンか、CDK の Stage を活用するパターンがありそうですが、どのようなケースで使い分けると良いのでしょうか？また、CDK の Stage を活用した実装で困った事はありますか？逆もあれば是非聞きたいです。

### スタック作成パターン

同じスタックを環境ごとに静的に作るパターン

```ts
new SampleStack(app, 'DevSampleStack', devProps);
new SampleStack(app, 'StgSampleStack', stgProps);
new SampleStack(app, 'PrdSampleStack', prdProps);
```

同じスタックを環境ごとに動的に作るパターン

```ts
const env: string = app.node.tryGetContext('env') ?? 'dev';
const props = getStackProps(env);
new SampleStack(app, 'SampleStack', props);

// 別ファイル(config.tsやparameter.ts)などで管理
const getStackProps = (env: string): SampleStackProps => {
  switch (env) {
    case 'dev':
      return {
        key: 'dev-key',
        env: {
          account: '111111111111',
          region: 'us-east-1',
        },
      };
    case 'stg':
      return {
        key: 'stg-key',
        env: {
          account: '222222222222',
          region: 'us-east-1',
        },
      };
    case 'prd':
      return {
        key: 'prd-key',
        env: {
          account: '333333333333',
          region: 'us-east-1',
        },
      };
    default:
      throw new Error(`Invalid environment: ${env}`);
  }
};
```

CDK の Stage を活用するパターン

```ts
export class MyStage extends Stage {
  constructor(scope: Construct, id: string, props?: MyStageProps) {
    super(scope, id, props);

    new SampleStack(this, 'SampleStack', props); // 実際はpropsをStackProps用に整形して渡す
  }
}

new MyStage(app, 'DevStage', devProps);
new MyStage(app, 'StgStage', stgProps);
new MyStage(app, 'PrdStage', prdProps);
```

### パターンの使い分け方

個人的には、 **「マルチスタック構成を環境ごとに、かつ静的に作りたい(呼びたい)場合」** にステージは便利です。

もしこのケースでステージがない場合、app.ts ではスタック数 × ステージ数の呼び出しが出てくるので、コードが複雑になって認知負荷が高まります。

```ts
// my-stage.ts
export class MyStage extends Stage {
  constructor(scope: Construct, id: string, props?: MyStageProps) {
    super(scope, id, props);

    new SampleStack1(this, 'SampleStack1', props); // 実際はpropsをStackProps用に整形して渡す
    new SampleStack2(this, 'SampleStack2', props); // 実際はpropsをStackProps用に整形して渡す
    new SampleStack3(this, 'SampleStack3', props); // 実際はpropsをStackProps用に整形して渡す
  }
}

// app.ts
new MyStage(app, 'DevStage', devProps);
new MyStage(app, 'StgStage', stgProps);
new MyStage(app, 'PrdStage', prdProps);
```

シングルスタックの場合、もしくは動的にスタックを作る場合は個人的にはステージは使いません。静的に作る場合で、今はシングルスタックだけど今後マルチスタックになるかもという予測や拡張性を担保しておきたい場合はあらかじめステージを作っておいても良いとは思います。

### CDK の Stage を活用した実装で困った事

### CDK の Stage を活用しなくて困った事

### CDK の Stage を活用して良かった事
