`na_ios/coredata`

`na_ios/coredata`は、扱うのに経験が必要なcoredataを、簡単に扱えるようにするモジュールです．

データの正しさとパフォーマンスの良さを両立するため、多くのAPIの内部ではスレッドを使い、なおかつ、それを隠蔽しています．

例えば、`TestObject`という`NSManagedObject`のクラスがあった場合、

```objective-c
[TestObject filter:@{@"name": @"test"} options:nil complete:^(NSArray *mos) {
    // 色々処理
}];
```

このように非同期メソッドの`complete`ハンドラに結果が渡されます．また`complete`ハンドラはmain threadで返ってくるため、ハンドラ内でUIの処理をしても、問題が無いようになっています．
非同期メソッドには`filter`の他に、`create`、`get`、`get_ore_create`などのAPIがあります．

```objective-c
[TestObject create:@{@"name": @"test2"} options:nil complete:^(id mo) {
	// hogehoge
}];

[TestObject get_or_create:@{@"name": @"test"} options:nil complete:^(id mo) {
	// hogehoge
}];
```

`create`や`get_or_create`はcontextに変更を加える可能性がありますが、その場合は、`TestObject`に登録した`mainContext`(main thread上のcontext)に変更がマージされてから`complete`ハンドラは呼ばれます．そのため、`complete`ハンドラ内でUIを更新すると、変更分も表示されることになります．

また、同じようにして、ハンドラを渡さない同期メソッドもあります．

```objective-c
TestObject *obj = [TestObject create:@{@"name": @"test"} options:nil];
NSArray *objs = [TestObject filter:@{@"name": @"test"} options:nil];
TestObject *obj2 = [TestObject get_or_create:@{@"name": @"test"} options:nil];
Bool bl = (obj == obj2); => YES
```

最後に、独自にcoredata上でスレッドを作成したい上級者向けには、次のようなメソッドがあります．

```objective-c
[mainContext performBlockOutOfOwnThread:^(NSManagedObjectContext *context){
    // !!!!!!ここでいろいろと変更を加える!!!!!!
    [context save:nil];
} afterSaveOnMainThread:^(NSNotification *note) {
    // !!!!!!終了処理!!!!!!
}];
```

`performBlockOutOfOwnThread:^(NSManagedObjectContext *context)block afterSaveOnMainThread:^(NSNotification *note)save`は、main thread上のcontextからしか呼び出すことを考慮していない事に注意して下さい．
また`block`ハンドラはmain threadじゃなく、`save`ハンドラはmainThreadであることにも注意して下さい．
`save`ハンドラは`mainContext`に変更がマージされた後に呼び出されるため、`save`ハンドラ内で（`mainContext`につなげてある）UIを更新することで、変更分も表示することができます．

これは次の処理のラッパーになっています．


```objective-c
NSManagedObjectContext *mainContext = [ModelController sharedController].mainContext;
NSPersistentStoreCoordinator *coordinator = mainContext.persistentStoreCoordinator;
NSManagedObjectContext *context = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
[context setPersistentStoreCoordinator:coordinator];
[[NSNotificationCenter defaultCenter] addObserverForName:NSManagedObjectContextDidSaveNotification object:context queue:nil usingBlock:^(NSNotification *note) {
    [mainContext performSelectorOnMainThread:@selector(mergeChangesFromContextDidSaveNotification:)
                                  withObject:note
                               waitUntilDone:YES];
    dispatch_async(dispatch_get_main_queue(), ^{
        // !!!!!!終了処理!!!!!!
    });
}];

[context performBlock:^{
    // !!!!!!ここでいろいろと変更を加える!!!!!!
    [context save:nil];
}];
```

# 設定方法

`hoge.modeld`の中に`TestObject`と`TestObject2`が入っている場合、AppDelegateなどで、次のように書いて下さい．

```objective-c
NAModelController *controller = [NAModelController createControllerByName:@"hoge" bundle:nil];
[controller addManagedObjectClasses:@[[TestObject class], [TestObject2 class]]];
```

これで晴れて`na_ios/coredata`の全ての機能を使う事が出来ます．

# `magicalpanda/MagicalRecord`との比較

同じような目的のモジュールに、[magicalpanda/MagicalRecord](https://github.com/magicalpanda/MagicalRecord)があります．`magicalpanda/MagicalRecord`の“Performing Core Data operations on Threads“の章も合わせて参照して下さい．

`magicalpanda/MagicalRecord`では、filteringやsortingにメリットがあります．`na_ios/coredata`に含まれていないような複雑なフェッチを行うことができます．
これに対して、`na_ios/coredata`では複雑なfilteringやsortingを介するフェッチには`NSFetchedResultsController`経由で行い、`NSArray`を介さない方法を推奨しています．これは、`UITableViewController`などと併用する場合、パフォーマンスとメモリの観点において都合が良いからです．

`NSFetchedResultsController`と`UITableViewController`を`na_ios/coredata`で使うには`na_ios/coredata_ui`を参照して下さい．

`na_ios/coredata`ではフェッチに自由度がない代わりに、フェッチやインサートに非同期のメソッドを持っています．これらを使う事でUIのブロックを防ぐことを念頭におきつつ、複雑なスレッドプログラミングを隠蔽することを目的にしています．

`magicalpanda/MagicalRecord`も分かりやすいAPIを持ったすばらしいモジュールです．上記の比較事項を念頭に入れて、プログラマはどちらのライブラリを選ぶかを選択することができます．

# スキーマレス・コアデータのススメ

`na_ios/coredata`では、次の理由から、スキーマレスな`NSManagedObject`を推奨しています．

 1. スキーマ変更によるアップデート時のmigration
 2. サーバサイドのスキーマ変更への追従
 3. json形式でのサーバとの同期

1,2は非常にめんどうな問題です．サーバ側にしろクライアント側にしろスキーマ変更によるマイグレーションはユーザ、開発者にとって非常に手間のかかる、バグの混入しやすい作業です．一般的なwebアプリケーションとは異なり、ネイティブアプリケーションの場合、アップデート時にしかスキーマを変更出来ません．また、今までの経験上、スキーマに定義されるフィールドの多くは、わざわざカラムに持つ必要の無いものです．
そこで、`NSManagedObject`のデータは一つの`NSDictionary`(JSON)として持ち、INDEXの用途にのみattributeを定義する方法を紹介します．

```objective-c
#import "NSManagedObject+json.h"
```

をimportするとNSManagedObjectにupdateByJSONというAPIを生やしてくれます．

```objective-c
SchemalessModel *obj = [SchemalessModel create:@{} options:nil];
NSDictionary *json = @{
	@"prop1": @"hoge", 
	@"subdoc": @{
		@"prop2": @"fuga"
	}
};
[obj updateByJSON:json];
```

こうすることで、`SchemalessModel`が`prop1`や`subdoc__prop2`というattributeを持っていた場合、そこに`@"hoge"`と`@"fuga"`をマッピングしてくれます．ただし、次に述べるように、マッピングはしなくても利用することが出来ます．attributeにする場合は、その値で検索やソートをしたい場合に限って下さい．
また`data`(デフォルト．変更化)というattributeに`updateByJSON`に渡した`json`そのものを格納しておいてくれます．

これを利用するのは次のようなイメージです．

```objective-c
SchemalessModel *obj = [SchemalessModel get:@{} options:nil];
[cell.textLabel setText:obj.data[@"prop1"]]
or
[cell.textLabel setText:obj.prop1]
```

このようにすることで、マイグレーションのコストを押さえるのに加えて、数多くのフィールドを削減して、コードをクリーンに保つことが出来るでしょう．

# 依存関係

依存元：**なし**  
依存先: **na_ios/coredata_ui**, **na_ios/coredata_sync**


# 補足情報 個々のパッケージ

## coredata/categories
コアデータに関わる各種クラスのカテゴリが入っているパッケージになります．

`NSFetchRequest`, `NSManagedObjectContext`, `NSPredicate`, `NSManagedObject`の4種類に対応しています．まずは`NSManagedObject`だけを使うのが良いでしょう．

#### `categories/NSManagedObject+na`

moにCRUD操作を生やすためのカテゴリ．blockによるcallbackを引数に持つものは非同期で、メインスレッドに返ってくきます．

#### `categories/NSFetchRequest+na`

#### `categories/NSManagedObjectContext+na`

contextにCRUD操作を生やすためのカテゴリ．`categories/NSManagedObject+na`で利用しています．

#### `categories/NSPredicate+na`

NSDictionaryか、NSArrayからPredicateを作成するもので、NSDictionaryの方は`@"%K == %@", key, val`で評価し、NSArrayの方は、評価式を順番に入れておくショートカットを持っています．基本的には上記の3つのクラスを用いてNSPredicateを直接は触らないようにするのが得策です．

## coredata/controllers

#### `controllers/NAModelController`  

modeldファイル(`hoge.modeld`)の名前(`hoge`)を`name`に設定して、`setup()`を呼べば、以下のことをやってくれます． 
 - coordinatorの作成  
 - bundleからの初期コピー（bundle内に`hoge.sqlite`ファイルを入れておけば、初期状態としてそちらを使う．でかいデータの時に便利）  
 - mainthread上のcontextの作成

また`destroyAndSetup`を呼ぶと、sqliteファイルを削除して、もう一度最初からやり直すことができます．

## coredata/models

#### `models/NSDictionaryTransformer`

dictionaryとsqlite内のバイナリを自動変換するクラス
