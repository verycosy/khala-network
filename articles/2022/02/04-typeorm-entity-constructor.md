# TypeORM에서 entity의 constructor 사용시 주의사항

TypeORM 공식문서의 [entity 페이지](https://typeorm.io/#/entities)에는 다음과 같은 문구가 있습니다.

    When using an entity constructor its arguments must be optional.
    Since ORM creates instances of entity classes when loading from the database,
    therefore it is not aware of your constructor arguments.

DB에서 데이터를 불러올 때, ORM이 entity 클래스의 인스턴스를 생성하기 때문에  
entity constructor의 인수는 반드시 optional 이어야 한다고 설명합니다.

<br>

예시로 이메일과 휴대폰 번호를 갖는 User Entity를 만들어보겠습니다.

```typescript
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  email: string;

  @Column()
  phoneNumber: string;
}
```

<br>

이런 entity가 있을 때 일반적으로 이메일과 휴대폰 번호를 인수로 갖는 생성자를 생각해볼 수 있습니다.

```typescript
@Entity()
export class User {
    constructor(email: string, phoneNumber: string) {
        this.email = email;
        this.phoneNumber = phoneNumber;
    }

    ...
}
```

<br>

하지만 공식문서의 주의사항과 달리 optional 하지 않아도 정상적으로 작동합니다!

```typescript
const user = new User('verycosy@kakao.com', '010-1111-2222');
await this.userRepository.save(user); // DB에 저장 완료
```

그 이유는 인스턴스를 생성하는 시점에서는 단순히 값을 대입하고만 있기 때문인데요,  
생성자 안에서 유효성 검사하는 상황을 가정해보겠습니다.

```typescript
@Entity()
export class User {
  constructor(email: string, phoneNumber: string) {
    if (!email.includes('@')) {
      throw new Error('잘못된 이메일입니다.');
    }

    this.email = email;
    this.phoneNumber = phoneNumber;
  }
}
```

이렇게 전달받은 값을 **참조**하게 되면 문제가 생깁니다.  
ORM이 User 인스턴스를 생성할 때, `email`과 `phoneNumber`는 값이 `undefined` 이기 때문에 참조 오류가 발생합니다.  
따라서 TypeORM에서는 인수를 통한 entity 객체는 정적 메소드를 활용하여 생성하는 편이 좋습니다.

<br>

```typescript
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  email: string;

  @Column()
  phoneNumber: string;

  static create(email: string, phoneNumber: string): User {
    const user = new User();
    user.email = email;
    user.phoneNumber = phoneNumber;

    return user;
  }
}
```

<br>

협업 과정에서 다른 팀원이 constructor를 통해 인스턴스 생성하는 걸 방지하고 싶다면  
private 키워드도 고려해볼 수 있습니다.

```typescript
@Entity()
export class User {
  private constructor() {}

  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  email: string;

  @Column()
  phoneNumber: string;

  static create(email: string, phoneNumber: string): User {
    const user = new User();
    user.email = email;
    user.phoneNumber = phoneNumber;

    return user;
  }
}

const user = new User(); // Error
```
