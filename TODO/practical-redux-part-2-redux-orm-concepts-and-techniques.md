> * 原文地址：[Practical Redux, Part 2: Redux-ORM Concepts and Techniques](http://blog.isquaredsoftware.com/2016/10/practical-redux-part-2-redux-orm-concepts-and-techniques/)
* 原文作者：[Mark Erikson](https://twitter.com/acemarke)
* 译文出自：[掘金翻译计划](https://github.com/xitu/gold-miner)
* 译者：
* 校对者：

# Practical Redux, Part 2: Redux-ORM Concepts and Techniques




_Useful techniques for using Redux-ORM to help manage your normalized state, part 2:  
In-depth examples of how I use Redux-ORM_

#### Series Table of Contents

Continuing on from the previous post about what Redux-ORM is and why you might want to use it, let’s look at the **core concepts of Redux-ORM, and how I actually use it in my own application**.

> **Note**: The code examples in this post are intended to demonstrate the general concepts and workflow, and probably won’t entirely run as-is. **See the [series introduction](http://blog.isquaredsoftware.com/2016/10/practical-redux-part-0-introduction/)** for info on the example scenarios and plans for demonstrating these ideas in a working example application later.

## Redux-ORM Core Concepts

Redux-ORM serves as an extremely useful abstraction layer over the normalized data in your Redux store. There’s a number of key concepts to understand when using it:

### Sessions

The Session class is used to interact with an underlying set of data. If you use the reducer generated by `schema.reducer()`, Redux-ORM will create a Session instance internally. Otherwise, you can create Session instances by calling `schema.from(entities)` (which creates a Session that will apply updates immutably), or `schema.withMutations(entities)` (which creates a Session that will directly mutate the provided data).

When a Session instance is created from source data, Redux-ORM creates temporary subclasses of the Model types available in the Schema, “binds” them to that session, and exposes the subclasses as fields on the Session instance. This means **it’s important that you always extract the Model classes you want to use from the Session instance and interact with those**, rather than using the versions you might have directly imported at the module level. If you’re writing your reducers as part of your Model classes, Redux-ORM passes the bound version of the current class in as the third argument, and the current Session instance as the fourth argument.

### Models

The Model instances returned by a Session really just **act as facades over the plain Javascript objects inside the store**. When a Model instance is requested, Redux-ORM generates property fields on the Model instances based on the keys in the underlying object, as well as the declared relations. These property fields define getters and setters that encapsulate the real behavior. Depending on what that field is, the getters will return the plain value from the underlying object, a new Model instance for a single relation, or a QuerySet instance for a collection relation. The underlying object can be accessed directly using `someModelInstance.ref`.

The use of properties and getters also means that **relations are not denormalized until you actually access those properties**. So, even an entity has a lot of relations, there shouldn’t be any additional expense to just retrieving a Model instance for that entity.

Internally, Redux-ORM uses a queue of actions, and applies those actions kind of like a mini-Redux (where each action is used to update its internal state reducer-style). For example, running `const pilot = Pilot.create(attributes); pilot.name = "Jaime Wolf";` would queue up a `CREATE` action and an `UPDATE` action. **Neither of those actions is applied until you call one of the appropriate methods on the session or model**, such as `session.reduce()`. There is one exception to this: if the session was created using `schema.withMutations(entities)`, it _will_ directly apply all updates to the affected object right away. Otherwise, **all updates are queued up, then applied immutably in sequence to produce the final result**.

### Managing Relations

Redux-ORM relies on a “QuerySet” class as an abstraction for managing collections. A QuerySet knows what Model type it relates to, and contains a list of IDs. Operations such as `filter()` return a new QuerySet instance with a different internal list of IDs. QuerySets have an internal flag indicating whether they should dereference just the plain objects, or corresponding Model instances. This is used by chaining the properties `withModels` or `withRefs` into the lookup process, such as `this.mechs.withModels.map(mechModel => mechModel.name)`.

For `many`-type relations, Redux-ORM will auto-generate “through model” classes, which store the IDs for both items in the relation. For example, a `Lance` with a field `pilots : many("Pilot")` would result in a `LancePilot` class and table. There’s a PR currently open to allow customization of those through models, to better enable situations like ordering of items in that relation.

### Synchronization

It’s also important to understand that Redux-ORM **_does not_** include any capabilities for synchronizing data with a server (such as the methods included with Backbone.Model or Ember Data). It is _only_ about managing relations stored locally as plain JS data. (In fact, despite the name, it doesn’t even depend on Redux at all.) You are responsible for handling data syncing yourself.

## Typical Usage

I primarily use Redux-ORM as a specialized “super-selector” and “super-immutable-update” tool. This means that I work with it in my selector functions, thunks, reducers, and `mapState` functions. Here’s some of the practices I’ve come up with.

### Entity Selection

Because I consistently use the Schema singleton instance across my application, I’ve created a selector that encapsulates extracting the current `entities` slice and returning a Session instance initialized with that data:

    import {createSelector} from "reselect";
    import schema from "./schema";

    export const selectEntities = state => state.entities;

    export const getEntitiesSession = createSelector(
        selectEntities,
        entities => schema.from(entities),
    );

Using that selector, I can retrieve a Session instance inside of a `mapState` function, and look up pieces of data as needed by that component.

This has a couple benefits. In particular, because many different `mapState` functions might be trying to do data lookups right in a row, only one Session instance should be created per store update, so this should be something of a performance optimization. Redux-ORM does offer a `schema.createSelector()` function that is supposed to create optimized selectors that track which models were accessed, but I haven’t yet gotten around to actually trying that. I may look into it later when I do some perf/optimization passes on my own application.

Overall, I make it a point to keep all my components unaware of Redux-ORM’s existence, and only pass plain data as props to my components.

### Writing Entity-Based Reducers

Most of my entity-related reducers are based more around a specific action case rather than a certain Model class. Because of this, some of my reducers are fairly generic, and take an item type and an item ID as parameters in the action payload. As an example, here’s a generic reducer for updating the attributes of any arbitrary Model instance:

    export function updateEntity(state, payload) {
        const {itemType, itemID, newItemAttributes} = payload;

        const session = schema.from(state);
        const ModelClass = session[itemType];

        let newState = state;

        if(ModelClass.hasId(itemID)) {
            const modelInstance = ModelClass.withId(itemID);

            modelInstance.update(newItemAttributes);

            newState = session.reduce();
        }

        return newState;
    }

Not all my reducers are this generic - some of them do end up specifically referencing certain model types in specific ways. In some cases, I can build up higher-level functionality by reusing these generic building block reducers.

My reducers generally follow the same pattern: extract parameters from payload, create Session instance, queue updates, apply updates using `session.reduce()`, and return the new state. There’s admittedly some verbosity there, which I could probably abstract out a bit more if I wanted to, but the overall consistency and the simplification of the actual update logic is a big win in my book.

I’ve also written a couple small utilities that aid with the process of looking up a given model by its type and ID:

    export function getModelByType(session, itemType, itemID) {
        const modelClass = session[itemType];
        const model = modelClass.withId(itemID);
        return model;
    }

    export function getModelIdentifiers(model) {
        return {
            itemID : model.getId(),
            itemType : model.getClass().modelName,
        };
    }

Many of my actions contain `itemType` and `itemID` pairs in their payloads. Part of that is that I personally lean towards keeping my actions fairly minimal, with more work in the thunks _and_ the reducers as necessary, and don’t like blindly merging data from an action straight into my state.

I’ve found that I frequently need to apply updates in a multi-step fashion. However, because the Model instances are facades over the plain data, this doesn’t always work well. If I’ve queued up some update (such as `someModel.someField = 123`), that change isn’t really “visible” to the Model instances until it’s been applied. Since updates are applied immutably, this situation can get complicated.

One way to handle this might be to create an initial Session instance with the starting data, then create a second Session instance with the updated data:

    const firstSession = schema.from(entities);
    const {Pilot} = firstSession;

    const pilot = Pilot.withId(pilotId);
    // Attribute update queued up here
    pilot.name = "Natasha Kerensky";

    const updatedEntities = firstSession.reduce();

    const secondSession = schema.from(updatedEntities);
    const {Pilot : Pilot2} = secondSession;

    // do something with the classes from the second session, which
    // are now acting as facades over the updated data object

I’m not a fan of that approach, though. It’s kind of ugly, and there could be confusion over which Session and Model classes I need to use at any moment.

I did some digging through the Redux-ORM source, and noted that a Session instance ultimately is a wrapper around the object stored as `this.state` internally. Since that field is public, we can interact with it. In particular, I realized that I could take an existing Session instance and update it to reference a new state object, without having to create a second Session instance:

    const session = schema.from(entities);
    const {Pilot} = session;

    const pilot = Pilot.withId(pilotId);
    pilot.name = "Natasha Kerensky";

    // Immutably apply updates, then point the session at the updated state object.
    session.state = session.reduce();

    // All field/model lookups now use the updated state object

This approach has allowed me to implement some fairly complex multi-step data updates, while still handling all the data in an immutable fashion.

Since this process does actually modify the current Session instance, I make it a point to never do this with the “shared” Session instance returned from my `getEntitiesSession()` selector. If I’m doing this in a reducer, I always create a new Session anyway. If I’m doing this in a thunk, I use a second selector to create a separate session instance for this task:

    export function getUnsharedEntitiesSession(state) {
        const entities = selectEntities(state);
        return schema.from(entities);
    }

## Adding Specialized Behavior

Redux-ORM provides some very useful tools for dealing with normalized data, but it only has so much functionality built-in. Fortunately, it also serves as a great starting point for building additional functionality.

### Serializing and Deserializing Data

As mentioned in the previous post, the Normalizr library is the de-facto standard for normalizing data received from the server. I’ve found that Redux-ORM can be used to mostly build a replacement for Normalizr. I added static `parse()` methods to each of my classes, which know how to handle the incoming data based on the relations:

    class Lance extends Model {
        static parse(lanceData) {
            // Because it's a static method, "this" refers to the class itself.
            // In this case, we're running inside a subclass bound to the Session.
            const {Pilot, Battlemech, Officer} = this.session;

            // Assume our incoming data looks like:
            // {name, commander : {}, mechs : [], pilots : []}

            let clonedData = {
               ...lanceData,
               commander = Officer.parse(clonedData.commander),
               mechs : lanceData.mechs.map(mech => Battlemech.parse(mech)),
               pilots : lanceData.pilots.map(pilot => Pilot.parse(pilot))
            };

            return this.create(clonedData);
        }
    }

This method could then be called in either a thunk or a reducer to help process data from a response, and queue up the necessary `CREATE` actions inside of Redux-ORM. If called in a reducer, the updates could be applied directly to the existing state. If used in a thunk, you’d want to take the resulting normalized data and include it in a dispatched action for merging into the store.

> **Note**: My own data is just nested, with no duplication. I had originally assumed that this process would handle duplicate data okay, by merging it together or something similar. However, some testing has shown that assumption was wrong. If an item with an ID already actually exists in the state, and a `CREATE` action with the same type and ID is queued, Redux-ORM will throw an error. If two `CREATE` actions with the same ID are queued up together, Redux-ORM will effectively throw away the first created entry and replace it with the second created entry. I’ve [opened up an issue](https://github.com/tommikaikkonen/redux-orm/issues/50) to discuss what the desired behavior should be.

On the flip side, it’s often necessary to send a denormalized version of the data back to the server. I’ve added `toJSON()` methods to my models to support that:

    class Lance extends Model {
        toJSON() {
            const data = {
                // Include all fields from the plain data object
                ...this.ref,
                // As well as serialized versions of all the relations we know about
                commander : this.commander.toJSON(),
                pilots : this.pilots.withModels.map(pilot => pilot.toJSON()),
                mechs : this.mechs.withModels.map(mech => mech.toJSON())
            };

            return data;
        }
    }

### Creation and Deletion

I often need to generate initial data for a new entity in an action creator before either sending it to the server or adding it to the store. I’m still waffling somewhat on what exactly is the best way to approach this. For now, I’ve added `generate` methods that can help encapsulate the process:

    const defaultAttributes = {
        name : "Unnamed Lance",
    };

    class Lance extends Model {
        static generate(specifiedAttributes = {}) {
            const id = generateUUID("lance");

            const mergedAttributes = {
                ...defaultAttributes,
                id,
                ...specifiedAttributes,     
            }

            return this.create(mergedAttributes);
        }
    }

    function createNewLance(name) {
        return (dispatch, getState) => {
            const session = getUnsharedEntitiesSession(getState());
            const {Lance} = session;

            const newLance = Lance.generate({name : "Command Lance"});
            session.state = session.reduce();
            const itemAttributes = newLance.toJSON();

            dispatch(createEntity("Lance", newLance.getId(), itemAttributes));
        }
    }

Meanwhile, Redux-ORM’s `Model.delete` method doesn’t fully cascade, so I’ve implemented a custom `deleteCascade` method where neded:

    class Lance extends Model {
        deleteCascade() {
            this.mechs.withModels.forEach(mechModel => mechModel.deleteCascade());
            this.pilots.withModels.forEach(pilotModel => pilotModel.deleteCascade());
            this.delete();
        }
    }

I’ve also implemented a number of additions to better handle copying data back and forth between various versions of a model instance (such as a “current” version vs a “WIP draft” version), which I’ll discuss in a later post on data editing approaches.

## Final Thoughts

Redux-ORM does introduce some additional concepts on top of Redux. Like any abstraction layer, it can be leaky at times, and you do need to understand what’s going on at the Redux level. That said, I’ve found that it really enables me to think about my data management at a higher level of abstraction. In addition, its ability to handle relations and simplify the actual logic for immutably updating my data is a real timesaver, and has made a lot of my code simpler in the process. It’s already served me very well, and I can’t wait to see what other improvements Tommi Kaikkonen might make in the future.

For links to further information, including documentation and articles, see **[Part 1: Redux-ORM Basics](http://blog.isquaredsoftware.com/2016/10/practical-redux-part-1-redux-orm-basics/)**.



