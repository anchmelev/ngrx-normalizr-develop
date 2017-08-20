# ngrx-normalizr
> Managing [normalized state](http://redux.js.org/docs/recipes/reducers/NormalizingStateShape.html) in [ngrx](https://github.com/ngrx/platform) applications, transparently.


This package provides a set of actions, reducers and selectors for handling normalization and denormalization of state data **transparently**.
*ngrx-normalizr* uses [normalizr](https://github.com/paularmstrong/normalizr) for [normalizing](https://github.com/paularmstrong/normalizr/blob/master/docs/api.md#normalizedata-schema) and [denormalizing](https://github.com/paularmstrong/normalizr/blob/master/docs/api.md#denormalizeinput-schema-entities) data. All normalization and denormalization
is defined by the use of [normalizr schemas](https://github.com/paularmstrong/normalizr/blob/master/docs/api.md#schema), since that's the way normalizr works. This enables selectors to use a transparent and powerful projection of state data.

## Installation
To install this package:
```sh
yarn add ngrx-normalizr
npm i ngrx-normalizr
```

### Peer dependencies
*ngrx-normalizr* defines [normalizr](https://github.com/paularmstrong/normalizr) and [@ngrx-store](https://github.com/ngrx/platform/blob/master/docs/store/README.md) as its peer dependencies, so you need to install them if not present already:

```sh
yarn add @ngrx/store normalizr
npm i @ngrx/store normalizr
```

## Usage
Also refer to the [Typedoc documentation](https://michaelkrone.github.io/ngrx-normalizr/).
To enable the normalizing reducer to store normalized data, you have to add it to your state. The best place for this might be the root state of your application, but feature states may use their own normalized state as well. Extend your state interface with the `NormalizedState` interface. This will
force the `ActionReducerMap` to implement a reducer which reduces the state to a `NormalizedState`.

```javascript
import { ActionReducerMap } from '@ngrx/store';
import { NormalizedState, reducer } from 'ngrx-normalizr';

export interface State extends NormalizedState {
  /* ... other state properties */
}

export const reducers: ActionReducerMap<State> = {
	'@@normalized': reducer,
  /* ... other state reducers */
};
```

If there are no other state properties, it is sufficient to add the *ngrx-normalizr* reducer to your state reducers:
```javascript
export const reducers: ActionReducerMap<NormalizedState> = {
	'@@normalized': reducer
};
```

Now you have a `@@normalized` state property which can hold the normalized data. Do not worry about the weird name,
you will not have to deal with it.

> *ngrx-normalizr* itself does not rely on any [Angular](https://angular.io) feature.

### Schemas
[Schemas](https://github.com/paularmstrong/normalizr/blob/master/docs/api.md#schema) define the relations of your data.
In order to normalize and denormalize data, normalizr needs to be feed with a schema. In this example, a user might have
an array of pets:
```javascript
import { schema } from 'normalizr';

export class Pet {
	id: string;
	name: string;
	type: 'cat' | 'dog';
}

export class User {
	id: string;
	name: string;
	pets: Pet[];
}

export const petSchema = new schema.Entity('pets');
export const userSchema = new schema.Entity('users', { pets: [petSchema] });
```

### Setting data
To add data and automatically normalize it, *ngrx-normalizr* provides a `SetData` action. This action takes an object with `data` and `schema` as a
payload. Entities are identified by their id attribute defined in the schema. Existing entities will be overwritten by the new data, other entities
will be added to the store.

###### Using `SetData` in an effect

```javascript
@Effect()
loadEffect$ = this.actions$
  .ofType(LOAD)
  .switchMap(action => this.http.get('https://example.com/api/user'))
  .mergeMap((result: User[]) => [
    // dispatch to add data to the store
    new SetData<User>({ data: result, schema: userSchema }),
    // dispatch to inform feature reducer
    new LoadSuccess(data)
  ])
  .catch(err => Observable.of(new LoadFail(err)));
```

### Removing data
To remove data, *ngrx-normalizr* provides a `RemoveData` action. This action takes an object with `id`, `schema` and an optional `removeChildren` property as payload. The schema entity with the given id will be removed. If `removeChildren` is a map of the schema key mapped to an object property, all referenced child entities will also be removed from the store.

###### Using `RemoveData` in an effect
```javascript
@Effect()
removeEffect$ = this.actions$
  .ofType(REMOVE)
  .switchMap((action: Remove) => this.http.delete(`http://example.com/api/user/${action.payload}`))
  .mergeMap(result => [
    // dispatch to remove data from the store
    new RemoveData({ id: result.id, schema: userSchema, removeChildren: { pets: 'pets' } }),
    // dispatch to inform feature reducer
    new RemoveSuccess()
  ])
  .catch(err => Observable.of(new RemoveFail(err)));
```

### Schema selectors
To transparently query data from the store from a feature module, you can use the selectors provided by the `createSchemaSelectors` function.
It takes an entity schema to create schema bound selectors:

```javascript
import { createSchemaSelectors } from 'ngrx-normalizr';
import { User } from '../classes/user';

const schemaSelectors = createSchemaSelectors<User>(userSchema);
```

`createSchemaSelectors` will return schema bound selectors:

* `getEntities` - ` MemoizedSelector<{}, T[]>` Returns all denormalized entities
* `getNormalizedEntities` - `MemoizedSelector<any, EntityMap>` Returns the all normalized (raw) state enities
* `entitiesProjector` - `(entities: {}) => T[]` Projector function for denormalizing a set of entities to an entity array
* `entityProjector` - `(entities: {}, id: string) => T` Projector function for denormalizing a single entity with the given id

You might create several selectors with several schemas, i.e. a *listView* schema, which only denormalizes the data used in the list
view, and a *detailView* schema, to completely denormalize a given entity.

#### Using schema selectors
Feature selectors can use the schema bound selectors and projector functions to query entity data from the store. To get all denormalized
entities, you might simply use the `getEntities` selector like this:

```javascript
// store.select(getUsers) will give you all denormalized user entities
const getUsers = schemaSelectors.getEntities;
```
Under the hood this does something similar to this:
```javascript
// equivalent alternative
const getUsers = createSelector(
  schemaSelectors.getNormalizedEntities,
  schemaSelectors.entitiesProjector
);

```
The `entitiesProjector` simply takes an object of normalized entity data and applies the denormalization with the bound schema.
To query and denormalize specific data you can use the *@ngrx/store* [`createSelectors`](https://github.com/ngrx/platform/blob/master/docs/store/selectors.md#createselector) function and compose them with the schema bound
selectors:

```javascript
export const getSelectedUserId = createSelector(
  userFeatureSelector,
  user.getSelectedId
);

// store.select(getSelectedUser) will give you the denormalized selected user
const getSelectedUser = createSelector(
  schemaSelectors.getNormalizedEntities,
  getSelectedUserId,
  schemaSelectors.entityProjector
);
```
`entityProjector` will simply take an object of denormalized entities and apply the denormalization with the bound schema only for the given id. Note that you might select data from the complete denormalized result, but this would force a denomalization of all schema bound entities:

```javascript
// alternative (denormalizing all user entities)
const getSelectedUser = createSelector(
  getUsers,
  getSelectedId,
  (entities, id) => entities.find(e => e.id === id)
);
```

## Meta

Michael Krone – [@DevDig](https://twitter.com/DevDig) – michael.krone@outlook.com

Distributed under the XYZ license. See ``LICENSE`` for more information.

[https://github.com/michaelkrone/ngrx-normalizr](https://github.com/michaelkrone/ngrx-normalizr)

## Contributing

1. Fork it (<https://github.com/michaelkrone/ngrx-normalizr>)
2. Create your feature branch (`git checkout -b feature/fooBar`)
3. Commit your changes (`git commit -am 'Add some fooBar'`)
4. Push to the branch (`git push origin feature/fooBar`)
5. Create a new Pull Request