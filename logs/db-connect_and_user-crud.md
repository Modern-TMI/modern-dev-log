# 로그인 / 회원가입

로그인과 회원가입을 구현하기 위해서는 다음과 같은 기능들을 필요로 한다.

* **DB**
  * User CRUD
* 인증 (Authentication)
* 권한 설정

## DB 연동

db는 rdbms의 mariadb를 사용한다.  
local 기반 환경에서 테스트 할 수 있도록 도커를 이용해 db를 설치한다.

```shell
$ docker pull mariadb   # mariadb 이미지 다운로드
$ docker container run mariadb \  # mariadb 이미지로 컨테이너 생성
  -d \  # Background 실행을 위한 옵션
  -p 3306:3306 \  # [host]:[container]. host와 container간의 port 연결
  -e MYSQL_ROOT_PASSWORD=ROOT_PASSWORD \  # mariadb의 환경변수. DB Root password 설정
  -e MYSQL_DATABASE=DATABASE_NAME \  # mariadb의 환경변수. default database를 설정
  --name mariadb  # container의 이름을 지정. 
```

혹은 `docker-compose.yml` 파일을 이용한다.

```yaml
version: "3.1"

services:
  database:
    image: mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ROOT_PASSWORD
      MYSQL_DATABASE: DATABASE_NAME
    ports:
      - 3306:3306
```

## User CRUD

### Install Package

Database와의 연결을 위해 nestjs에서는 `typeorm`을 사용한다.  
필요로 하는 패키지를 설치해준다.

```shell
$ npm i @nestjs/typeorm typeorm mysql2
```

### Typeorm Config Setting

Database와의 연결을 위해 `app.module.ts`에 Typeorm과 관련된 내용을 추가해준다.

```typescript
// app/app.module.ts

@Module({
    imports: [
        TypeOrmModule.forRoot({
            host: 'localhost',
            port: 3306,
            type: 'mariadb',
            username: 'root',
            password: 'ROOT_PASSWORD',
            database: 'DATABASE_NAME',
            entities: [],
            synchronize: true,
            autoLoadEntities: true,
            logging: true
        })
    ],
    controllers: [AppController],
    providers: [AppService]
})
export class AppModule {}
```

### User Entity 생성

> `Entity`는 Database 테이블에 매핑되는 클래스

Typeorm 에서는 Database 테이블과 맵핑을 위해 `Entity`를 만들어 줘야 한다.  
User Entity를 만들기 이전에, User와 관련된 로직을 묶어서 처리할 `Module`을 만들어 준다

```shell
$ nx g @nrwl/nest:module app/users -p modern-tmi-server
$ nx g @nrwl/nest:service app/users -p modern-tmi-server
$ nx g @nrwl/nest:controller app/users -p modern-tmi-server
```

`app/` 디렉토리 아래에 `users.module.ts` 파일이 만들어 지면 `users.entity.ts` 파일을 만들어 준다.

```typescript
// app/users/users.entity.ts

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  email: string;

  @Column()
  isActive: boolean;

  @Column()
  nickname: string;

  @Column()
  password: string;

  @CreateDateColumn()
  createDate: Date;

  @UpdateDateColumn()
  updatedDate: Date;
}
```

`Entity` 클래스는 `@Entity()` 데코레이터를 이용한다.  
Database의 Column은 `@Column()`을 사용해 표현 할 수 있다.  
`@PrimaryGeneratedColumn()`이나 `@CreateDateColumn()`등 Column에 대한 데코레이터에 대한 자세한 정보는 [링크](https://typeorm.io/#/entities) 를 참고!

`Entity`를 사용(CRUD)하기 위해서는 Entity를 관리하는 Repository를 만들어야 한다.

`users.repository.ts` 파일을 만들어 `User` Entity를 연결해주자.

```typescript
// app/users/users.repository.ts

@EntityRepository(User)
export class UsersREpository extends Repository<User> {}
```


### User Entity 적용

위에서 만든 Entity를 이용해 실제 Table을 만들기 위해서는 `TypeOrmModule`에 연결해야 한다.

`users.module.ts`에 다음과 같은 내용을 추가한다

```typescript
// app/users/users.module.ts

@Module({
    imports: [
        TypeOrmModule.forFeature([UsersRepository])
    ],
    exports: [UsersService, TypeOrmModule],
    controllers: [UsersController],
    providers: [UsersService]
})
```

`imports` 에 `TypeOrmModule.forFeature()`에 `UsersRepository`를 연결하면 TypeOrm에서 자동으로 Entity를 로드해준다.

> 자동으로 로드하는 이유는, 위의 Typeorm Config 에서 `autoLoadEntities`를 true로 해주었기 때문!
> `autoLoadEntities`의 기본값은 `false`

### User CRUD Service 작성

CRUD를 하기 위해 다음과 같은 사전 작업을 진행했다

1. Entity 생성
2. Entity를 사용하기 위한 Repository 생성

이제는 본격적으로 CRUD를 위한 비즈니스 로직을 작성한다.

모든 비즈니스 로직은 Layer 분리를 위해 `users.service.ts`에 작성된다.

```typescript
// app/users/users.service.ts

@Injectable()
export class UsersService {
    constructor(private usersRepository: UsersRepository) {}
    
    findAll() {
        return this.usersRepository.find();
    }
    
    findOne(id: number) {
        return this.usersRepository.findOne(id);
    }
    
    createUser(userData: any) {
        const user = new User();
        user.email = userData.email;
        user.password = userData.password;
        user.nickname = userData.nickname;
        
        return this.usersRepository.save(user);
    }
    
    updateUser(id: number, data: any) {
        // updateColumnName에 수정할 column명을 작성
        // 단일 column만 변경하는 경우 PUT 보다는 PATCH에 더 가까움
        return this.usersRepository.update({id}, {updateColumnName: data})
    }
    
    deleteUser(id: number) {
        return this.usersRepository.delete({id});
    }
}
```

### User Controller 작성

작성한 UsersService는 Controller를 통해서 호출 해준다

```typescript
// app/users/users.controller.ts

@Controller('users')
export class UsersController {
    constructor(private usersService: UsersService) {}
    
    @Get()
    getUsers() {
        return this.usersService.findAll();
    }
    
    @Get(':id')
    getUser(@Param('id') id: number) {
        return this.usersService.findOne(id);
    }
    
    @Post()
    createUser(@Body() userData: any) {
        return this.usersService.createUser(userData);
    }
    
    @Put(':id')
    updateUser(@Param('id') id: number, @Body() updateData: any) {
        // 단일 column만 변경하는 경우 PUT 보다는 PATCH에 더 가까움
        return this.usersService.updateUser(id, updateData);
    }
    
    @Delete(':id')
    deleteUser(@Param('id') id: number) {
        return this.usersService.deleteUser(id);
    }
}
```

위의 Controller는 `@Contoller()` 데코레이터에 정의된 `/users` 라우트 그룹 아래에서 Users와 관련된 로직을 처리 할 수 있다.

### User Controller 연결

작성된 Controller는 AppModule에 등록되어야 라우팅에 연결되어 접근할 수 있다.

```typescript
// app/app.module.ts

@Module({
    imports: [
        TypeOrmModule.forRoot({}),
        UsersModule // 추가!
    ],
    controllers: [AppController],
    providers: [AppService]
})
export class AppModule {}
```

이후 각각의 CRUD를 위해선 아래의 URL로 접근 할 수 있다.

```http request
### 모든 유저 가져오기
GET http://localhost/users

### 특정 유저 가져오기
GET http://localhost/users/1

### 유저 생성하기
POST http://localhost/users

{
  "email": "ktojong@gmail.com",
  "password": "password",
  "nickname": "nickname"
}

### 유저 수정하기
PUT http://localhost/users/1

// Update Body는 샘플.. 

{
  "password": "newPassword",
}

### 유저 삭제하기
DELETE http://localhost/users
```
