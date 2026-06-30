Source: https://sbc-knowledge.bsscommerce.com/doc/coding-conventions-QjmAcI6pZF

# 1. Tổng quan

`**Core principles Frontend Backend**`

`**Shopify Security Code Review**`

4 pillars of code quality

* Clean & Readable
* Consistent
* Secure
* Performance

# 2. Core principles

## 2.1. Phạm vi áp dụng

Các nguyên tắc và quy tắc này áp dụng cho **Frontend**, **Backend**, và Shopify.

## 2.2. Các nguyên tắc nền tảng

* **KISS (Keep It Simple, Stupid)**:
  * Đặt tên (biến, hàm, class,...) rõ ràng, tự mô tả bàn thân nó làm gì, ưu tiên dùng `const` hơn `let`, tránh dùng `var`


* **Fail Fast**: Validate input sớm và return hoặc throw error ngay lập tức để tránh nested quá sâu

  ```javascript
  class EmailValidator {
      async execute(email: string) {
          if (!email) throw new Error("empty email");
          if (email === "email@example.com") throw new Error("bad email");
          return email;     
      }
  }
  ```
* **Single Responsibility Principle (SRP)**: Một function, component, hoặc file chỉ nên làm một việc và làm tốt việc đó.

  ```jsx
  // Bad example: Mixing responsibility in a single component
  function UserProfile() {
    const [user, setUser] = React.useState(null);
  
    React.useEffect(() => {
      fetch('/api/user')
        .then(res => res.json())
        .then(setUser);
    }, []);
  
    if (!user) return <p>Loading...</p>;
  
    return (
      <div>
        <h1>{user.name}</h1>
      </div>
    );
  }
  
  // Good example Separating concerns into presentational and container components
  const useUser = () =>
    useQuery(['user'], () =>
      fetch('/api/user').then(res => res.json())
    );
  
  // Component
  const UserProfile = ({ user }) => (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
  
  const UserProfileContainer = () => {
    const { data: user, isLoading } = useUser();
  
    if (isLoading) return <p>Loading...</p>;
    return <UserProfile user={user} />;
  };
  
  
  ```
* **Dependency Inversion Principle (DIP):** Module cấp cao không nên phụ thuộc vào module cấp thấp. Cả hai nên phụ thuộc vào abstraction

  ```javascript
  // WITHOUT Dependency Inversion - TIGHT COUPLING
  class SequelizeShopRepository {
      findOne(domain) {}
      findMany(query) {}
  }
  
  class ShopService {
      private shopRepository;
      constructor() {
          this.shopRepository = new SequelizeShopRepository();
      }
      
      getShop(domain) {
          return this.shopRepository.findOne(domain);
      }
  }
  // const shopService = new ShopService();
  
  
  // WITH Dependency Inversion - LOOSE COUPLING
  class ShopRepository {
      findOne(domain) { throw new Error("Method not implemented.") }
      findMany(query) { throw new Error("Method not implemented.") }
  }
  
  class SequelizeShopRepository extends ShopRepository {
      constructor() {
          super();
      }
      findOne(domain) {
          //
      }
      findMany(query) {
          //
      }
  }
  
  class ShopService {
      private shopRepository;
      constructor(shopRepository) {
          this.shopRepository = shopRepository;
      }
      
      getShop(domain) {
          return this.shopRepository.findOne(domain);
      }
  }
  // const shopRepository = new SequelizeShopRepository();
  // const shopService = new ShopService(shopRepository);
  ```

  ```jsx
  // Bad example: Mixing responsibility in a single component
  const UserProfile = () => {
    const { data } = useQuery(['user'], () =>
      fetch('/api/user').then(res => res.json())
    );
  
    return <div>{data ? data.name : 'Loading...'}</div>;
  };
  
  // Good example Separating concerns into presentational and container components
  // UserProfileContainer.js
  export const useUserProfile = (userService) => {
    return useQuery(['user'], () => userService.getUser());
  };
  
  // Component
  const UserProfile = ({ userService }) => {
    const { data, isLoading } = useUserProfile(userService);
    return <div>{isLoading ? data?.name : 'Loading...'}</div>;
  };
  ```
* Tránh over-engineering: nghĩa là không làm hệ thống, code hoặc giải pháp quá phức tạp hơn mức cần thiết để đáp ứng yêu cầu thực tế.
* Refactor khi cần
* Chia nhỏ bài toán

# 3. Frontend

**Phạm vi áp dụng hiện tại**: React + Javascript/ Typescript

## 3.1. Naming conventions

| **Element** | **Convention** | **Examples** |
|---------|------------|----------|
| Component | PascalCase | `CPTable`, `UserProfile` |
| Hook    | useCamelCase | `useShop`, `useProducts` |
| Handler | handleCamelCase | `handleClick`, `handleSubmit` |
| File    | Tùy        | `CPTable.jsx`, `useShop.js` |
| Boolean | is/has/should | `isLoading`, `hasPermission` |
| Component Event | onCamelCase | `onProfileChange` |

## 3.2. Formatting & Linting

* **ESLint** là bắt buộc (không ignore rule nếu không có lý do chính đáng)
* Thứ tự import:

 ![](/api/attachments.redirect?id=8b564c43-2cfa-443a-85da-df324b8380ac " =942x118")

* Kiểm soát export trong các file index tránh circular dependency và bundle tool build các đoạn code k dùng hoặc k cần thiết (theo kinh nghiệm thì không nên re export từ index khác)
* ### Không nên `import * as` nếu k cần thiết điều này gây nên bundler build các đoạn code k cần thiết gây phình to size của app
* ### Lazy-load các module hoặc component nặng bằng dynamic import, chỉ khi thực sự cần.

```typescript
const MyComponent = React.lazy(() => import('./MyComponent'));
```

```typescript
function handleClick() {
  import('./math').then(math => {
  });
}
```

## 3.3. Comments & Documentation

* **Chỉ sử dụng comment khi cần thiết**
* Comment cần giải thích:
* * Business rules
  * Các hạn chế
  * Workaround hoặc trade-offs
* Tránh comments thừa
* Đối với các utility hoặc hook phức tạp, sử dụng **JSDoc** ngắn gọn.

  ![](/api/attachments.redirect?id=458b3789-eb29-4645-a3e7-5146b4b4f8f1 " =955x390")

## 3.4. Best Practices

* **Composition**: Ưu tiên các component nhỏ, tập trung vào một nhiệm vụ duy nhất.
* **Business Logic**: Chuyển **business logic** vào **custom hooks** hoặc **services**.\nBằng cách sử dụng HOC (higher-order component) hoặc custom hook,  có thể mở rộng hành vi của Component mà không cần sửa đổi cấu trúc ban đầu.

  ```typescript
  // Logger HOC that wraps a component
  const withLogger = (WrappedComponent) => {
    return (props) => {
      console.log("Component Rendered with Props: ", props);
      return <WrappedComponent {...props} />;
    };
  };
  // Usage
  const MyComponent = (props) => <div>{props.content}</div>;
  export default withLogger(MyComponent);
  ```
* **Performance**: Chỉ sử dụng useMemo và useCallback cho tính toán nặng hoặc để ngăn re-render tốn kém của child components.
* **Shopify Polaris:**
* * Tuân thủ chặt chẽ, và nghiêm ngặt [__Shopify Polaris__](https://polaris.shopify.com/).
  * Tránh custom styling trừ khi Polaris không thể hỗ trợ được.
* **Ví dụ**

```jsx
// ❌ Bad: Logic clutter, cryptic naming, and non-Polaris styling
const d2 = ({ p }) => {
  const [l, setL] = useState(false);
  if (p) {
    return <div style={{color: 'red'}}>Error</div>
  }

  return <div>{/*...*/}</div>
}


// ✅ Good: Clear naming, Early returns, Polaris components, Business logic in Hooks

import { Banner, Page } from "@shopify/polaris";

const ProductManagement = () => {
  const { products, isLoading, error } = useProductList(); // Custom hook for logic

  if (error) {
    return <Banner status="critical">Failed to load products</Banner>;
  }

  return (
    <Page title="Products">
      {/* Polaris Layout Components */}
    </Page>
  );
};
```

# 4. Backend

**Phạm vi áp dụng hiện tại**: [Node.js](http://node.js) + KoaJS/NestJS

## 4.1. Naming Conventions

| **Element** | **Convention** | **Examples** |
|---------|------------|----------|
| Folder  | kebab-case | `draft-order`, `user-profile` |
| Variable | camelCase  | `draftOrder`, `userProfile` |
| Class   | PascalCase | `OrderService`, `UserModel` |

## 4.2. Request & Error Handling Flows

 ![](/api/attachments.redirect?id=50dbbb62-f4f5-4c6f-9e76-d307526dbc62 " =892x730")

 ![](/api/attachments.redirect?id=1a05514a-997a-4e73-944a-6a785f3f93bb " =1279x326")

## 4.3. Best Practices

* **Controller-service pattern**:
  * Controllers xử lý HTTP routing và input validation.
  * Services xử lý business logic.
  * Không bao giờ đặt DB query trong controllers.
  * Trong một số trường hợp, pattern controller-service-repository-model sẽ rõ ràng hơn.
* **Idempotency**: Đảm bảo các thao tác quan trọng (ví dụ: tạo charge hoặc refund) có thể retry mà không gây ra side-effect.
* **Environment agnostic**: Sử dụng config service để quản lý environment variables; tránh sử dụng `process.env` trực tiếp trong business logic.
* **Logging**: Sử dụng logger có cấu trúc rõ ràng. Mỗi log phải bao gồm request_id phục vụ việc truy vết.
* Tránh query N + 1 không kiểm xoát, đảm bảo số lượng query N + 1 phải không được loop quá nhiều phần tử cũng như liên tục vào DB

  ```typescript
  ❌ // 1 query lấy tất cả users
  const users = await userRepository.find(); 
  
  // 1 query cho mỗi user để lấy posts
  for (const user of users) {
    user.posts = await postRepository.find({ where: { userId: user.id } });
  }
  
  ✅ // Query Eager Loading / Join
  const users = await userRepository.find({ relations: ['posts'] });
  
  ```
* **Shopify GraphQL**:
* * **Standard**: Sử dụng GraphQL thay vì REST cho tất cả Shopify APIs. Tìm hiểu thêm tại [__REST Admin API reference__](https://shopify.dev/docs/api/admin-rest).\n![](/api/attachments.redirect?id=5f217f85-aaea-4c4d-ae94-f0d7bcab2c5a " =669x83")
  * **Fragments**: Sử dụng GraphQL fragments để giữ query dễ bảo trì và tái sử dụng.
  * **Throttling**: Sử dụng thư viện nội bộ [__bss-sbc/shopift-api-fetcher__](https://www.npmjs.com/package/@bss-sbc/shopify-api-fetcher) hoặc tự triển khai logic retry tự động cho lỗi 429 Too Many Requests.\nTìm hiểu sâu hơn tại [__Shopify API limits__](https://shopify.dev/docs/api/usage/limits).
* **Ví dụ**

```javascript
// src/controllers/shop.controller.js
const { BadRequestException, NotFoundException } = require("@/shared/http");
const logger = require("@/logger");

class ShopController {
    constructor(shopService) {
        this.shopService = shopService;
    }

    async getShopByDomain(ctx) {
       
        
    }
}

module.exports = ShopController;

// src/services/shop.service.js
class ShopService {
    constructor(shopRepo) {
        this.shopRepo = shopRepo;
    }

    async findOneByDomain(domain, options = {}) {
        const shop = await this.shopRepo.findOneByDomain(domain, options);
        return {
            ...shop.toJSON()
        }
    }
}

module.exports = ShopService;

// src/repositories/shop.repo.js
class IShopRepository {
    findOneByDomain(domain, options = {}) {
        throw new Error('Not implemented');
    }
}

class SequelizeShopRepository extends IShopRepository {
    constructor(shopModel) {
        super();
        this.Shop = shopModel;
    }

    findOneByDomain(domain, options = {}) {
        return this.Shop.findOne({
            where: { domain },
            ...options
        });
    }
}

module.exports = {
    IShopRepository,
    SequelizeShopRepository
}

// src/routes/shop.route.js
const Router = require("@koa/router");
// Phần này implement Dependency Injection (DI) thủ công
// Có thể triển khai DI bằng thư viện,
//    hoặc tự implement,
//    hoặc Factory pattern,
//    hoặc gom hết vào modules/instances directory (giống NestJS)
const ShopController = require("@/controllers/shop.controller");
const ShopService = require("@/services/shop.service");
const { SequelizeShopRepository } = require("@/repositories/shop.repo");
const Shop = require("@/models/shop.model");

const shopRouter = new Router("/shop");
const shopRepository = new SequelizeShopRepository(Shop);
const shopService = new ShopService(shopRepository);
const shopController = new ShopController(shopService);

shopRouter.get("/:domain", (ctx) => shopController.getShopByDomain(ctx));

module.exports = shopRouter;
```

# 5. Security & Data Privacy

## 5.1. Secrets Management

* **Environment variables:** Lưu trữ tất cả các API key và secret trong .env hoặc secret manager. 
* **Nghiêm cấm commit** .env vào Version Control (hiện tại là Git).

## 5.2. PII (Personally Identifiable Information)

* **Data masking**: Che giấu dữ liệu nhạy cảm của khách hàng (ví dụ email, số điện thoại) trong log.

## 5.3. Dependencies

- [ ] **Audit:** Chạy npm audit hàng tuần


 ![](/api/attachments.redirect?id=aea261d2-bf05-4711-82bb-03fab13a9928 " =385x320")

- [ ] **Strict Versions:** Sử dụng version chính xác cho package quan trọng; tránh `^` hoặc `~` cho thư viện Shopify để ngăn breaking change trong auto-update
- [ ] **Tree Shaking:** Chỉ import function cần thiết từ thư viện lớn (ví dụ, `import { map } from 'lodash'` thay vì `import _ from 'lodash'`).

# 6. Code Review Checklist

 ![](/api/attachments.redirect?id=05a7a8a9-4994-464d-8986-e966876f4e39 " =958x508")

Trước khi approve Merge Requests, reviewer **phải** kiểm tra:

- [ ] **Conventions:** Code đã tuân thủ conventions chưa?
- [ ] **Logic:** Có đáp ứng yêu cầu Jira không? Các edge case (null/undefined) đã được xử lý?
- [ ] **Performance:** Có N+1 query trong GraphQL hoặc loop không? Có useEffect trigger không cần thiết không?
- [ ] **Security:**  Input đã được validate? PII đã được ẩn trong log?
- [ ] **Polaris:** UI có khớp với Shopify Design System không?
- [ ] **(Optional) Test:** Có unit test cho logic phức tạp không? CI build có pass không?
