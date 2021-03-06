## How to setup project

- project 配下で init
  `$ expo init`
  → よしなに設定

- 起動 → ブラウザの開発ツールが立ち上がる
  `$ expo start`
  → ブラウザから run ios/Android simulator でシミュレーターが立ち上がる

- インストールしている端末を確認
  `emulator -list-avds`

### 注意

立ち上げるシミュレーターに `-dns-server 8.8.8.8`をつける

### ファイルの共有方法

`$ expo publish`
→ よしなに URL が発行される →QR を読み込むと自分の端末で確認できる

- ios の場合はクライアントと共有する場合、同じアカウントでログインする必要がある

- app.json に privacy: unlisted を追加すると公開されない（他の人に）

## core components

https://reactnative.dev/docs/components-and-apis

### firebase プロジェクト作成

</>で web app 作成(reactNative なので)
よしなに設定 →firebase を利用するために自分のプロジェクトに firebase Config を追加する

- env.js と適当に作ってそこに入れる

```js
const firebaseConfig = {
  apiKey: '…',
  authDomain: '…',
  projectId: '…',
  storageBucket: '…',
  messagingSenderId: '…',
  appId: '…',
};
```

Firebase プロジェクトの作成

```
$ npm install firebase
```

- App.jsx に追加

```js
import firebase from 'firebase';
import firebaseConfig from 'env';
if (firebase.apps.length === 0) {
  firebase.initializeApp(firebaseConfig);
}
```

### Authentication から email とパスワードから認証を有効にする

<img width="1089" alt="スクリーンショット 2021-07-17 13 03 33" src="https://user-images.githubusercontent.com/69241625/126025983-0c8cde95-5697-4d76-9019-983a6bb9da9c.png">
- Sign UP と Login 部分を実装
  firebase の auth メソッドから createUserWithEmailAndPassword と signInWithEmailAndPassword を使う

```js
const handlePress = () => {
  firebase
    .auth()
    .createUserWithEmailAndPassword(email, password)
    .then((userCredential) => {
      const { user } = userCredential;
      console.log(user.uid);
      navigation.reset({
        index: 0,
        routes: [{ name: 'MemoList' }],
      });
    })
    .catch((err) => Alert.alert(err.message));
};
```

- Login 状態の監視
  マウント時に user 情報があるかどうか

```js
useEffect(() => {
  const unsubscribe = firebase.auth().onAuthStateChanged((user) => {
    if (user) {
      navigation.reset({
        index: 0,
        routes: [{ name: 'MemoList' }],
      });
    }
  });
  return unsubscribe;
}, []);
```

### useEffect について

```js
useEffect(callback); //propsが変更されたり、画面がアップデートされるたびにcallbackが実行される
useEffect(callback, []); //コンポーネントがマウントされたときに一度だけcallbackが実行される
useEffect(callback, [foo]); //fooが更新されたらcallbackが実行される
```

- return して cleanup する(コンポーネントがアンマウントされる直前に cleanup が実行される)

```js
useEffect(() => {
  console.log('test'); //マウント時に実行
  return () => {
    console.log('Unmount!'); //アンマウント時に実行(直前)
  };
}, []);
```

### firebase の Database に情報を保存する(Cloud fireStore)

- [Cloud Firestore のデータモデル](https://firebase.google.com/docs/firestore/data-model?hl=ja)

`collection > document > data`
`referenceを指定するとdbから引っ張ってこれる`

- 使い方
- App.jsx に加える

```js
require('firebase/firestore');
```

- page

```js
const handlePress = () => {
  const { currentUser } = firebase.auth(); //loginしているユーザ
  const db = firebase.firestore(); //firestore呼び出す
  const ref = db.collection(`users/${currentUser.uid}/memos`); //userごとのmemoのref
  ref //refに対して入力値をadd
    .add({
      bodyText,
      updatedAt: new Date(),
    })
    .then((docRef) => {
      console.log('created', docRef.id);
    })
    .catch((err) => {
      console.log('Error!', err);
    });
};
```

- db からデータを取り出す

```js
useEffect(() => {
  const db = firebase.firestore();
  const { currentUser } = firebase.auth();
  const ref = db.collection(`users/${currentUser.uid}/memos`);
  const unsubscribe = ref.onSnapshot((snapshot) => {
    snapshot.forEach((doc) => {
      console.log(doc.data());
    });
  });
  return unsubscribe; //監視を解除
}, []);
```

```
Object {
  "bodyText": "あああ",
  "updatedAt": Object {
    "nanoseconds": 710000000,
    "seconds": 1626571433,
  },
}
```

- レンダリングの高速化
  画面に表示されている部分のみを表示する

```jsx
<FlatList
  data={memos}
  renderItem={renderitem} //viewの部分はまとめる
  keyExtractor={(item) => item.id} //デフォルトでketを探しにいくので明示的に指定する
/>
```

### snapshot について

[document](https://firebase.google.com/docs/firestore/query-data/listen?hl=ja)

```js
db.collection('foo').onSnapshot(() => {}); //複数のドキュメントスナップショットを含む

db.collection('foo') //単一のドキュメントデータを持っている
  .doc('bar')
  .onSnapshot(() => {});
```

### Appstore GooglePlay store に出す

- 画像が必要(splash, icon 等) -> assets に入れる
- app.json 修正(画像の PATH, ios, android への bundleIdentifier 等の追加)

### ビルドする(IOS)

- simulator 上でビルド後の動作確認

```sh
$ expo whoami ->ログインしてたらOK
$ expo build: ios

? Choose the build type you would like: › - Use arrow-keys. Return to submit.
   archive - Deploy the build to the store
❯  simulator - Run the build on a simulator

```

-> URL 発行/login して確認(ビルドは結構時間かかりそう)

-> 解答して simulator にドラックアンドドロップして動作確認

- 公開する(archive を選ぶ)
  よしなに設定

->ビルド終わったらダウンロード .ipa(apple store に配信する用)
Transporter を使ってビルドしたファイルを appstore にあげる

### ビルドする(Android)

```
$ expo build:Android

-> generate key storeする

$ expo fetch:android:keystore
->どこかに.jskファイルを移動させておく

```
