# MongoDB Aggregate

A _flexible_ programatic interface for MongoDB aggregations with simplicity and
test-ability in mind. Allows for fluid assembly and post-assembly manipulation
of an aggregation pipeline.

**Basic example (simplest conversioon):**

```ts
  const cursor = aggregate(db.collection('orders'), ($) => [
    $.match({ orderStatus: $.in('new', 'pending' }),
    $.unwind('$orderItems'),
    $.replaceRoot('$orderItems'),
    $.match({ quantity: $.gt(0) }),
    $.lookup(db.collection('products'), 'product', 'productId', '_id'),
    $.unwind('$product', true),
    $.group('$product.category', {
      productsSold: $.tally(),
      itemsSold: $.sum('$quantity'),
      amountSold: $.sum('$itemTotal'),
    }),
    $.sort({ itemsSold: -1 }),
  ]);
  ...
  // The pipeline is built when the cursor executed! :)
  const $results = await cursor.toArray();
```
The Pipeline object is exposed via the "cursor" and can be manipulated prior to
execution. Eg.

```ts
  // Alter the match criteria in the first match stage
  cursor.pipeline.getStage('match').orderStatus = 'complete';
  // remove the second match stage
  cursor.pipeline.dropStage('match', 1);
```

**The Goal**

* Normalize and declare inputs that affect the aggregation pipeline
* Simplified integration testing

```js
// file: ./db/agg/product-search.ts
const { Aggregation } = require('mongodb-aggregate');

export Aggregation.create(function () {
  this.search('KEYWORDS')
      .project({ title: 1, categoryId: 1 });
      .sort({ searchScore: '$$$SORT_DIRECTION' });
  this.phase('joinCategory')
    .lookup('categories', 'category', 'categoryId', '_id')
    .unwind('category', true);
  this.paginate('$$$PAGE_NUM', '$$$PER_PAGE');
})
  .target('myArticles')
  .collections({ categories: 'myCategoriesCollection' })
  .variables({
    KEYWORDS: { type: 'string', required: true },
    CATEGORIES: { string: true, array: true, default: [] },
    SORT_TYPE: { enum: ['relevance', 'popularity'], default: 'relevance' },
    SORT_DIRECTION: 'number',
  }),
  .setup(function (input) {
    const numCats = input.CATEGORIES.length;
    if (numCats) {
      this.searchPhase.match.categoryId: numCats === 1
          ? input.CATEGORIES[0]
          : $.in(input.CATEGORIES);
    }
    if (input.SORT_TYPE === 'popularity') {
      this.sortStage.unset('searchScore').set('viewCount', '$$$SORT_DIRECTION');
    }
  }),
  scope({
    mapCategoryInputToId: (val
  });
});
```

```ts
// file: db/agg/product-search.spec.ts
```

## Examples

#### Named Aggregation Stages

```ts
import aggregate from 'mongodb-aggregate';
import Pipeline from 'mongodb-aggregate/pipeline';
import { stageName } from 'mongodb-aggregate/annotations';

(() => {
  const pipeline = Pipeline.factory(function () => [
    this.match({ orderStatus: $.in('new', 'pending') }, 'matchOrders')),
    this.unwind('$orderItems'),
    this.match({ 'orderItems.quantity': $.gt(0) } }, 'matchOrderItems')),
    this.lookup(db.collection('products'), 'orderItems.productId', '_id')
    this.replaceRoot('$orderItems');
    this.group('$category', {
      itemsSold: $.sum('$quantity'),
      amountSold: $.sum('$itemTotal'),
    }, 'groupOrderItems'),
    this.addFields({ _id: '$$REMOVE', category: '$_id' }, 'alterResult');
    this.sort({ itemsSold: -1 }, 'sortResults');
  ]);

  // Retroactive named-access to manipulate aggregation 
  pipeline.getStageByName('matchOrders').orderStatus = 'sold';

  // Override sort
  pipeline.getStageByName('sortResults').unset('itemsSold').set('amountSold', -1);

  // Remove a stage altogether
  pipeline.removeStageByName('matchOrders');

  const results = await aggregate(db.collection('products'), pipeline);
})();
```
