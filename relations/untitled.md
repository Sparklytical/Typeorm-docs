# Relations FAQ

## How to create self referencing relation

Self-referencing relations are relations which have a relation to themself. This is useful when you are storing entities in a tree-like structures. Also "adjacency list" pattern is implemented using self-referenced relations. For example, you want to create categories tree in your application. Categories can nest categories, nested categories can nest other categories, etc. Self-referencing relations come handy here. Basically self-referencing relations are just regular relations that targets entity itself. Example:

```typescript
import {Entity, PrimaryGeneratedColumn, Column, ManyToOne, OneToMany} from "typeorm";

@Entity()
export class Category {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    title: string;

    @Column()
    text: string;

    @ManyToOne(type => Category, category => category.childCategories)
    parentCategory: Category;

    @OneToMany(type => Category, category => category.parentCategory)
    childCategories: Category[];

}
```

## How to use relation id without joining relation

Sometimes you want to have in your object id of the related object without loading it. For example:

```typescript
import {Entity, PrimaryGeneratedColumn, Column} from "typeorm";

@Entity()
export class Profile {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    gender: string;

    @Column()
    photo: string;

}
```

```typescript
import {Entity, PrimaryGeneratedColumn, Column, OneToOne, JoinColumn} from "typeorm";
import {Profile} from "./Profile";

@Entity()
export class User {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @OneToOne(type => Profile)
    @JoinColumn()
    profile: Profile;

}
```

When you load a user without `profile` joined you won't have any information about profile in your user object, even profile id:

```javascript
User {
  id: 1,
  name: "Umed"
}
```

But sometimes you want to know what is the "profile id" of this user without loading the whole profile for this user. To do this you just need to add another property to your entity with `@Column` named exactly as the column created by your relation. Example:

```typescript
import {Entity, PrimaryGeneratedColumn, Column, OneToOne, JoinColumn} from "typeorm";
import {Profile} from "./Profile";

@Entity()
export class User {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @Column({ nullable: true })
    profileId: number;

    @OneToOne(type => Profile)
    @JoinColumn()
    profile: Profile;

}
```

That's all. Next time you load a user object it will contain a profile id:

```javascript
User {
  id: 1,
  name: "Umed",
  profileId: 1
}
```

## How to load relations in entities

The easiest way to load your entity relations is to use `relations` option in `FindOptions`:

```typescript
const users = await connection.getRepository(User).find({ relations: ["profile", "photos", "videos"] });
```

Alternative and more flexible way is to use `QueryBuilder`:

```typescript
const user = await connection
    .getRepository(User)
    .createQueryBuilder("user")
    .leftJoinAndSelect("user.profile", "profile")
    .leftJoinAndSelect("user.photos", "photo")
    .leftJoinAndSelect("user.videos", "video")
    .getMany();
```

Using `QueryBuilder` you can do `innerJoinAndSelect` instead of `leftJoinAndSelect` \(to learn the difference between `LEFT JOIN` and `INNER JOIN` refer to your SQL documentation\), you can join relation data by a condition, make ordering, etc.

Learn more about [`QueryBuilder`](select-query-builder.md).

## Avoid relation property initializers

Sometimes it is useful to initialize your relation properties, for example:

```typescript
import {Entity, PrimaryGeneratedColumn, Column, ManyToMany, JoinTable} from "typeorm";
import {Category} from "./Category";

@Entity()
export class Question {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    title: string;

    @Column()
    text: string;

    @ManyToMany(type => Category, category => category.questions)
    @JoinTable()
    categories: Category[] = []; // see = [] initialization here

}
```

However in TypeORM entities it may cause problems. To understand the problem, let's first try to load a Question entity WITHOUT the initializer set. When you load a question it will return an object like this:

```javascript
Question {
    id: 1,
    title: "Question about ..."
}
```

Now when you save this object `categories` inside it won't be touched - because it is unset.

But if you have initializer, the loaded object will look like as follow:

```javascript
Question {
    id: 1,
    title: "Question about ...",
    categories: []
}
```

When you save the object it will check if there are any categories in the database bind to the question - and it will detach all of them. Why? Because relation equal to `[]` or any items inside it will be considered like something was removed from it, there is no other way to check if an object was removed from entity or not.

Therefore, saving an object like this will bring you problems - it will remove all previously set categories.

How to avoid this behaviour? Simply don't initialize arrays in your entities. Same rule applies to a constructor - don't initialize it in a constructor as well.

