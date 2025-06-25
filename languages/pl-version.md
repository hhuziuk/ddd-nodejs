# Domain-Driven Design w Node.js

**Autor:** Heorhii Huziuk ‹[huziukwork@gmail.com](mailto:huziukwork@gmail.com)›
*Node.js Developer (Software Engineer / Backend Developer w FarmFusion)*

> **Warunki wstępne:**
> Do napisania tego artykułu skłoniło mnie niedostateczne omówienie tematu projektowania zorientowanego na domenę (Domain-Driven Design) w Node.js. Istnieje wiele indywidualnych podejść, które są niepopularne: każdy autor wnosi coś swojego i często są one ze sobą sprzeczne. Chociaż istnieje ogólna koncepcja DDD, nie wszystkie jej praktyki są niezbędne do tworzenia oprogramowania właśnie w środowisku Node.js; sytuacja jest podobna do wzorców projektowych `Gang Of Four`, których opis ma jedynie pośredni związek ze światem programowania w JavaScript. Życzę miłej lektury i serdecznie zapraszam do edycji lub dyskusji :)

---

### Spis treści

1.  [**Domena (Domain)**](#domena-domain)
    *   1.1 [Encje (Entities)](#encje-entities)
    *   1.2 [Obiekty Wartości (Value Objects)](#obiekty-wartości-value-objects)
    *   1.3 [Agregaty (Aggregates)](#agregaty-aggregates)
    *   1.4 [Fabryki (Factories)](#fabryki-factories)
2.  [**Aplikacja (Application)**](#aplikacja-application)
    *   2.1 [Serwisy (Services)](#serwisy-services)
    *   2.2 [DTO Aplikacji (Application DTOs)](#dto-aplikacji-application-dtos)
    *   2.3 [Struktura (Aplikacja)](#struktura-aplikacja)
3.  [**Infrastruktura (Infrastructure)**](#infrastruktura-infrastructure)
    *   3.1 [Baza Danych (Database)](#baza-danych-database)
        *   3.1.1 [Mappery (Mappers)](#mappery-mappers)
    *   3.2 [Repozytoria (Interfejsy)](#repozytoria-interfejsy)
    *   3.3 [Repozytoria (Implementacje)](#repozytoria-implementacje)
    *   3.4 [Struktura (Infrastruktura)](#struktura-infrastruktura)
4.  [**Prezentacja (Presentation)**](#prezentacja-presentation)
    *   4.1 [Kontrolery / Trasy (Controllers / Routes)](#kontrolery--trasy-controllers--routes)
    *   4.2 [DTO (Prezentacja)](#dto-prezentacja)
    *   4.3 [Struktura (Prezentacja)](#struktura-prezentacja)
5.  [**Wyjątki (Exceptions)**](#wyjątki-exceptions)
6.  [**Podsumowanie**](#podsumowanie)
7.  [**Odnośniki**](#odnośniki)

---

### Domena (Domain)
> Warstwa Domeny (Domain) odpowiada za logikę biznesową. Czym jest logika biznesowa? - To zbiór zasad, według których działa aplikacja: lista pól, reguły walidacji itp.

#### Encje (Entities)
> W Encjach opisywane są pola i obiekty aplikacji. Obiekty w sensie: mamy użytkownika - musimy stworzyć dla niego encję, jest Taryfa - trzeba stworzyć dla niej encję. Encja - to jak tabela w bazie danych, ma określony zestaw pól: podstawowe to ID, imię, nazwisko, enum z rolami, kiedy rekord został utworzony itd.

Ważne uwagi:
 * Encja powinna hermetyzować zachowanie lub niezmienniki (prywatne, a nie publiczne).
 * Encja nie sprawdza stanu, nie sprawdza formatu e-maila ani poprawności telefonu.
 * Encja powinna zawierać obiekty wartości (value-objects), jeśli mamy złożone dane: telefon, e-mail, czas...


Oto przykład, jak można zaimplementować encję dla użytkownika:
```ts
import { Email } from '../value-objects/Email'; // tutaj użyto VO, zostanie on omówiony w następnym rozdziale

export class User {
    private readonly id: string;
    private name: string;
    private email: Email;
    private isActive: boolean;

    constructor(id: string, name: string, email: Email, isActive: boolean){
      this.id = id;
      this.name = name;
      this.email = email;
      this.isActive = isActive;
    }

    getEmail(): Email {
      return this.email;
    }

    changeEmail(newEmail: Email): void {
      this.email = newEmail;
    }

    activate(): void {
      this.isActive = true;
    }

    deactivate() {
      this.isActive = false;
    }

    getName(): string{
      return this.name;
    }

    toJson() {
      return {
        id: this.id,
        name: this.name,
        email: this.email.getValue(),
      }
  }
}
```

oto przykład użycia w serwisie aplikacyjnym:
```ts
function registerUser(name: string, rawEmail: string): User {
  const email = new Email(rawEmail); // Obiekt Wartości (Value Object): hermetyzuje walidację
  const user = new User(uuidv4(), name, email, false);
  return user;
}

// wyobraźmy sobie, że to przyszło z serwera: 'Olena', 'olena@example.com'
const user = registerUser('Olena', 'olena@example.com');
```

#### Obiekty Wartości (Value Objects)

> Następną częścią warstwy domenowej są Obiekty Wartości (Value Objects) - to faktycznie część, która realizuje dostęp do bardziej złożonych pól: Hasło, Email, NumerTelefonu. gdzie musimy przeprowadzać pewne sprawdzenia lub walidację zgodnie z zdefiniowanymi przez nas regułami biznesowymi.

Główne właściwości, które określają odrębność Obiektów Wartości:
 * hermetyzuje wartość
 * gwarantuje ich poprawność zgodnie z regułami biznesowymi
 * nie ma tożsamości(na odróżnienie od Encji)
 * porównywany według wartości, a nie identyfikatora

```ts
// email.vo.ts
export class Email {
  private readonly value: string;

  constructor(value: string) {
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
      throw new Error('Invalid email');
    }
    this.value = value;
  }

  getValue(): string {
    return this.value;
  }
}
```

```ts
// password.vo.ts
import * as bcrypt from 'bcrypt';

export class Password {
  private readonly hashed: string;

  private constructor(hashed: string) {
    this.hashed = hashed;
  }

  static async create(raw: string): Promise<Password> {
    if (raw.length < 8) throw new Error('Password too short');
    const hashed = await bcrypt.hash(raw, 10);
    return new Password(hashed);
  }

  static fromHashed(hashed: string): Password {
    return new Password(hashed);
  }

  async compare(raw: string): Promise<boolean> {
    return bcrypt.compare(raw, this.hashed);
  }

  getHashed(): string {
    return this.hashed;
  }
}
```

Przykład użycia w Encji:

```ts
import { Email } from '../value-objects/email.vo';
import { Password } from '../value-objects/password.vo';

export class User {
  constructor(
    private readonly id: string,
    private name: string,
    private surname: string,
    private email: Email,
    private phone: string | null,
    private password: Password,
    private isActive: boolean,
    private readonly createdAt: Date,
  ) {}

  getId(): string { return this.id; }
  getEmail(): string { return this.email.getValue(); }
  getHashedPassword(): string { return this.password.getHashed(); }

  activate() { this.isActive = true; }
  deactivate() { this.isActive = false; }

  async comparePassword(raw: string): Promise<boolean> {
    return this.password.compare(raw);
  }

  toPrimitives() {
    return {
      id: this.id,
      name: this.name,
      surname: this.surname,
      email: this.email.getValue(),
      phone: this.phone,
      password: this.password.getHashed(),
      isActive: this.isActive,
      createdAt: this.createdAt
    };
  }
}
```

Oto inny przykład użycia:

```ts
// money.vo.ts
export class Money {
  private readonly amount: number;
  private readonly currency: string;

  constructor(amount: number, currency: string) {
    if (amount < 0) throw new Error('Amount cannot be negative');

    const allowedCurrencies = ['USD', 'EUR', 'UAH'];
    if (!allowedCurrencies.includes(currency)) {
      throw new Error(`Unsupported currency: ${currency}`);
    }

    this.amount = amount;
    this.currency = currency;
  }

  getAmount(): number {
    return this.amount;
  }

  getCurrency(): string {
    return this.currency;
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) throw new Error('Currency mismatch');
    return new Money(this.amount + other.amount, this.currency);
  }

  subtract(other: Money): Money {
    if (this.currency !== other.currency) throw new Error('Currency mismatch');
    const result = this.amount - other.amount;
    if (result < 0) throw new Error('Resulting amount cannot be negative');
    return new Money(result, this.currency);
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  toString(): string {
    return `${this.amount.toFixed(2)} ${this.currency}`;
  }
}
```

I przykład użycia w encji domenowej i repozytorium:

Encja:
```ts
// src/domain/entities/invoice.entity.ts
import { Money } from '../value-objects/money.vo';
import { v4 as uuidv4 } from 'uuid';

export class Invoice {
  private readonly id: string;
  private readonly customerId: string;
  private total: Money;
  private isPaid: boolean = false;

  constructor(customerId: string, total: Money, id?: string) {
    this.id = id ?? uuidv4();
    this.customerId = customerId;
    this.total = total;
  }

  getId(): string {
    return this.id;
  }

  getTotal(): Money {
    return this.total;
  }

  markAsPaid(): void {
    this.isPaid = true;
  }

  applyDiscount(discount: Money): void {
    this.total = this.total.subtract(discount);
  }

  toJSON() {
    return {
      id: this.id,
      customerId: this.customerId,
      total: this.total.toString(),
      isPaid: this.isPaid,
    };
  }
}
```

Mała encja ORM:
```ts
// src/infrastructure/database/invoice.orm-entity.ts
export class InvoiceOrmEntity {
  id: string;
  customerId: string;
  amount: number;
  currency: string;
  isPaid: boolean;
}
```

A oto samo repozytorium:
```ts
// src/infrastructure/repositories/invoice.repository.ts
import { InvoiceOrmEntity } from '../database/invoice.orm-entity';
import { Invoice } from '../../domain/entities/invoice.entity';
import { Money } from '../../domain/value-objects/money.vo';

export class InvoiceRepository {
  private db: InvoiceOrmEntity[] = []; // umowna baza danych w pamięci

  async save(invoice: Invoice): Promise<void> {
    const record: InvoiceOrmEntity = {
      id: invoice.getId(),
      customerId: invoice['customerId'],
      amount: invoice.getTotal().getAmount(),
      currency: invoice.getTotal().getCurrency(),
      isPaid: invoice['isPaid'],
    };
    this.db.push(record);
  }

  async findById(id: string): Promise<Invoice | null> {
    const found = this.db.find(i => i.id === id);
    if (!found) return null;

    return new Invoice(
      found.customerId,
      new Money(found.amount, found.currency),
      found.id,
    );
  }
}
```

Serwis:
```ts
// src/application/invoice.service.ts
import { Money } from '../domain/value-objects/money.vo';
import { Invoice } from '../domain/entities/invoice.entity';
import { InvoiceRepository } from '../infrastructure/repositories/invoice.repository';

export class InvoiceService {
  constructor(private readonly repo: InvoiceRepository) {}

  async createInvoice(customerId: string, rawAmount: number, currency: string): Promise<Invoice> {
    const total = new Money(rawAmount, currency);
    const invoice = new Invoice(customerId, total);
    await this.repo.save(invoice);
    return invoice;
  }

  async applyDiscount(invoiceId: string, discountValue: number): Promise<Invoice> {
    const invoice = await this.repo.findById(invoiceId);
    if (!invoice) throw new Error('Invoice not found');

    const discount = new Money(discountValue, invoice.getTotal().getCurrency());
    invoice.applyDiscount(discount);
    await this.repo.save(invoice);
    return invoice;
  }
}
```

#### Agregaty (Aggregates)
> Agregaty - to zbiór wzajemnie powiązanych obiektów, które system traktuje jako jedną całość na potrzeby aktualizacji i zapisu.

##### **Podstawowe pojęcia**
1. **Korzeń Agregatu (Aggregate Root)**
    - jest jedynym „punktem wejścia” do interakcji z agregatem.
    - zapewnia spójność wszystkich obiektów agregatu.
    - tylko korzeń agregatu jest przechowywany w repozytorium, a wewnętrzne obiekty są modyfikowane za pomocą metod korzenia.
2. **Kontekst Ograniczony (Bounded Context)**
    - agregaty istnieją w granicach określonego kontekstu.
    - w jednym kontekście korzeń jednego agregatu nie odwołuje się do wewnętrznych obiektów innego — interakcja odbywa się za pomocą identyfikatorów lub zdarzeń domenowych.
3. **Niezmienniki Agregatu**
    - reguły i warunki, które zawsze muszą być spełnione dla stanu wewnątrz agregatu.

Oto schemat przedstawiający przykład interakcji dwóch agregatów:

![Aggregate Schema](../images/aggregate-schema-1.png)

*Schemat 1: Schemat interakcji agregatów. Źródło: [Domain-Driven Aggregates Explained | Why you should use them](https://youtu.be/SvnsOX4oVVo?si=fOr35OPCDUUFNWQn)*

##### **Dlaczego potrzebne są agregaty?**
* **Spójność:** zapewniają, że wszystkie części agregatu zawsze znajdują się w poprawnym stanie po każdej operacji.
* **Granica transakcyjna:** wszystkie zmiany w agregacie są zatwierdzane jako jedna transakcja.
* **Izolacja:** chronią wewnętrzną strukturę (hermetyzacja), pozwalając na dostęp tylko przez korzeń.
* **Przejrzystość modelu:** klastry obiektów są logicznie połączone w jedną całość według kontekstu biznesowego.

Oto schemat, który obrazowo pokazuje, jaką pajęczynę relacji otrzymujemy, nie tworząc agregatów:
![No-aggregate Schema](../images/aggregate-schema-2.png)

*Schemat 2: Brak agregatów w kodzie. Źródło: [Domain-Driven Aggregates Explained | Why you should use them](https://youtu.be/SvnsOX4oVVo?si=fOr35OPCDUUFNWQn)*

Oto schemat demonstrujący przejrzystość, jaką daje nam użycie agregatów:

![Aggregate Schema](../images/aggregate-schema-3.png)

*Schemat 3: Kod podzielony na agregaty. Źródło: [Domain-Driven Aggregates Explained | Why you should use them](https://youtu.be/SvnsOX4oVVo?si=fOr35OPCDUUFNWQn)*

##### **Użycie agregatów w kodzie**

Przykład agregatu `Product`, gdzie implementujemy `Stock` jako encję, a Price jako `Value Object`:

```ts
// Value Object: Price
export class Price {
  private readonly _amount: number;
  private readonly _currency: string;

  constructor(amount: number, currency: string) {
    if (amount < 0) {
      throw new Error("Amount cannot be a negative number");
    }
    if (!currency || currency.trim().length !== 3) {
      throw new Error("Currency must be a valid 3-letter code");
    }
    this._amount = amount;
    this._currency = currency.toUpperCase();
  }

  get amount(): number {
    return this._amount;
  }

  get currency(): string {
    return this._currency;
  }

  toString(): string {
    return `${this._amount} ${this._currency}`;
  }
}

// Value Object: Location
export class Location {
  private readonly _longitude: number;
  private readonly _latitude: number;

  constructor(longitude: number, latitude: number) {
    if (longitude < -180 || longitude > 180) {
      throw new Error("Longitude must be between -180 and 180");
    }
    if (latitude < -90 || latitude > 90) {
      throw new Error("Latitude must be between -90 and 90");
    }
    this._longitude = longitude;
    this._latitude = latitude;
  }

  get longitude(): number {
    return this._longitude;
  }

  get latitude(): number {
    return this._latitude;
  }

  toString(): string {
    return `(${this._latitude}, ${this._longitude})`;
  }
}

// Entity: Stock (część agregatu Product)
export class Stock {
  private _location: Location;
  private _quantity: number;

  constructor(location: Location, quantity: number) {
    if (quantity < 0) {
      throw new Error("Quantity cannot be negative");
    }
    this._location = location;
    this._quantity = quantity;
  }

  get location(): Location {
    return this._location;
  }

  get quantity(): number {
    return this._quantity;
  }

  increase(amount: number): void {
    if (amount <= 0) {
      throw new Error("Increase amount must be positive");
    }
    this._quantity += amount;
  }

  decrease(amount: number): void {
    if (amount <= 0 || amount > this._quantity) {
      throw new Error("Invalid decrease amount");
    }
    this._quantity -= amount;
  }
}

// Aggregate Root: Product
export class Product {
  private readonly _id: string;
  private readonly _price: Price;
  private readonly _weight: number;
  private readonly _stocks: Stock[];

  constructor(id: string, price: Price, weight: number, stocks: Stock[]) {
    if (!id || id.trim() === "") {
      throw new Error("Product id must be provided");
    }
    if (weight <= 0) {
      throw new Error("Weight must be a positive number");
    }
    if (!stocks || stocks.length === 0) {
      throw new Error("At least one stock location is required");
    }

    this._id = id;
    this._price = price;
    this._weight = weight;
    this._stocks = stocks;
  }

  get id(): string {
    return this._id;
  }

  get price(): Price {
    return this._price;
  }

  get weight(): number {
    return this._weight;
  }

  get stocks(): ReadonlyArray<Stock> {
    return this._stocks;
  }

  totalQuantity(): number {
    return this._stocks.reduce((sum, s) => sum + s.quantity, 0);
  }

  findStockByLocation(loc: Location): Stock | undefined {
    return this._stocks.find(
      s =>
        s.location.latitude === loc.latitude &&
        s.location.longitude === loc.longitude
    );
  }

  addStock(location: Location, quantity: number): void {
    if (this.findStockByLocation(location)) {
      throw new Error("Stock already exists for this location");
    }
    if (quantity < 0) {
      throw new Error("Quantity cannot be negative");
    }
    this._stocks.push(new Stock(location, quantity));
  }

  transferStock(from: Location, to: Location, quantity: number): void {
    const src = this.findStockByLocation(from);
    const dst = this.findStockByLocation(to);

    if (!src) throw new Error("Source location not found");
    if (!dst) throw new Error("Destination location not found");

    src.decrease(quantity);
    dst.increase(quantity);
  }
}
```

A oto przykład implementacji agregatu `Order`:
```ts
import { Product } from "./Product"; 

// Entity inside Order aggregate: OrderLineItem
export class OrderLineItem {
  private readonly _product: Product;
  private _quantity: number;

  constructor(product: Product, quantity: number) {
    if (quantity <= 0) {
      throw new Error("Quantity must be greater than zero");
    }
    this._product = product;
    this._quantity = quantity;
    this.ensureWeightLimit();
  } 

  get product(): Product {
    return this._product;
  }

  get quantity(): number {
    return this._quantity;
  }

  get weight(): number {
    return this._product.weight * this._quantity;
  }
  
  changeQuantity(newQuantity: number): void {
    if (newQuantity <= 0) {
      throw new Error("Quantity must be greater than zero");
    }
    this._quantity = newQuantity;
    this.ensureWeightLimit();
  }

  private ensureWeightLimit(): void {
    if (this.weight > 100) {
      throw new Error(
        `OrderLineItem weight (${this.weight}kg) exceeds 100kg limit`
      );
    }
  }
}

// Aggregate Root: Order
export class Order {
  private readonly _id: string;
  private readonly _lineItems: OrderLineItem[] = [];

  constructor(id: string, lineItems: OrderLineItem[] = []) {
    if (!id || id.trim() === "") {
      throw new Error("Order id must be provided");
    }
    this._id = id;
    lineItems.forEach((li) => this.addLineItem(li));
    this.ensureTotalWeightLimit();
  }

  get id(): string {
    return this._id;
  }

  get lineItems(): ReadonlyArray<OrderLineItem> {
    return this._lineItems;
  }

  get totalWeight(): number {
    return this._lineItems.reduce((sum, li) => sum + li.weight, 0);
  }

  addLineItem(item: OrderLineItem): void {
    if (this._lineItems.some((li) => li.product.id === item.product.id)) {
      throw new Error("Order already contains this product");
    }
    this._lineItems.push(item);
    this.ensureTotalWeightLimit();
  }

  removeLineItemByProductId(productId: string): void {
    const idx = this._lineItems.findIndex((li) => li.product.id === productId);
    if (idx === -1) {
      throw new Error("LineItem not found for productId " + productId);
    }
    this._lineItems.splice(idx, 1);
  }

  changeLineItemQuantity(productId: string, newQuantity: number): void {
    const li = this._lineItems.find((li) => li.product.id === productId);
    if (!li) {
      throw new Error("LineItem not found for productId " + productId);
    }
    li.changeQuantity(newQuantity);
    this.ensureTotalWeightLimit();
  }

  private ensureTotalWeightLimit(): void {
    if (this.totalWeight > 100) {
      throw new Error(
        `Order total weight (${this.totalWeight}kg) exceeds 100kg limit`
      );
    }
  }
}
```

#### Fabryki (Factories)
> W DDD wyróżnia się również fabryki - wyspecjalizowane obiekty lub metody odpowiedzialne za tworzenie złożonych obiektów domenowych w prawidłowym, spójnym stanie. Główną ideą jest przeniesienie całej „ciężkiej” logiki składania i walidacji wewnętrznych niezmienników z konstruktora do osobnej klasy lub metody, zamiast rozbudowywać same encje.

Do implementacji fabryki stosuje się dwa podejścia:

1. **Statyczna metoda fabrykująca (Static factory method)**

Metoda wewnątrz samego agregatu/encji:

```ts
export class Order {
  private constructor(/* ... */) { /* ... */ }

  public static createNew(
    customerId: string,
    items: OrderLineItem[],
  ): Order {
    // tutaj sprawdzenia, walidacja, ustawianie wartości domyślnych
    if (items.length === 0) {
      throw new Error("Order must have at least one item");
    }
    const order = new Order(/* … */);
    return order;
  }
}

// Wywołanie:
const order = Order.createNew(customerId, items);
```

2. **Dedykowana klasa fabryki (Dedicated factory class)**

Oddzielny serwis-fabryka:

```ts
export class OrderFactory {
  constructor(
    private readonly productRepo: ProductRepository,
    private readonly stockRepo: StockRepository
  ) {}

  public async createOrder(
    customerId: string,
    dtoItems: { productId: string; qty: number }[]
  ): Promise<Order> {
    // ładujemy produkty, sprawdzamy zapasy, tworzymy OrderLineItem…
    const items: OrderLineItem[] = [];
    for (const dto of dtoItems) {
      const product = await this.productRepo.getById(dto.productId);
      items.push(new OrderLineItem(product, dto.qty));
    }
    return Order.createNew(customerId, items);
  }
}

// Wywołanie z serwisu lub kontrolera:
const order = await orderFactory.createOrder(userId, payload.items);
```

##### Kiedy używać
* Gdy konstruktor agregatu/encji przestaje być „cienki” i nabiera dużo kodu walidacyjnego.
* Gdy trzeba wstrzykiwać zewnętrzne serwisy lub repozytoria podczas tworzenia.
* Gdy chcemy wyraźnie oddzielić obowiązki: fabryka — za tworzenie, agregat — za logikę biznesową po utworzeniu.


---

### **Aplikacja (Application)**
> Ta warstwa jest przeznaczona do implementacji przypadków użycia (use-case), a mianowicie do orkiestracji interakcji logiki biznesowej z Domeną (Encje, Obiekty Wartości, Agregaty) oraz światem zewnętrznym (interfejs użytkownika, API, infrastruktura itp.).

#### **Serwisy (Services)**
Serwisy Aplikacyjne (Application Services) - to serwisy, które:
    * nie zawierają logiki biznesowej
    * wywołują obiekty domenowe, agregaty, repozytoria
    * zarządzają transakcjami, jeśli jest to potrzebne
    * realizują przypadki użycia - wykonanie scenariusza biznesowego, na przykład rejestrację lub znalezienie użytkownika po ID, nie implementując logiki biznesowej, a jedynie wywołując obiekty domenowe, które tę logikę implementują

**Przykład serwisu:**
```ts
// application/services/register-user.service.ts
export class RegisterUserService {
  constructor(
    private readonly userRepo: IUserRepository,
    private readonly logger: ILogger,
  ) {}

  async execute(request: RegisterUserRequest): Promise<RegisterUserResponse> {
    this.logger.log(`Registering user: ${request.email}`, 'RegisterUserService');

    const email = new Email(request.email); // VO
    const user = new User(uuidv4(), request.name, email, false); // Entity

    await this.userRepo.save(user);

    this.logger.log(`User registered: ${user.getEmail().getValue()}`, 'RegisterUserService');

    return new RegisterUserResponse(user.getId(), user.getEmail().getValue());
  }
}
```


#### **DTO Aplikacji (Application DTOs)**
Obiekty Transferu Danych (Data Transfer Objects):
 * opisują dane wejściowe i wyjściowe dla serwisów
 * adaptują dane do/z domeny
 * są używane do izolacji od zewnętrznych modeli (encji ORM lub żądań HTTP)

**Przykład DTO:**
```ts
// dto/register-user.request.ts
export class RegisterUserRequest {
  constructor(public name: string, public email: string) {}
}

// dto/register-user.response.ts
export class RegisterUserResponse {
  constructor(public id: string, public email: string) {}
}
```

Interakcja `RegisterUserService` w kontrolerze będzie wyglądać mniej więcej tak:
```ts
@Post('/register')
async register(@Body() body: RegisterUserDto) { // nasze pierwsze DTO
  const req = new RegisterUserRequest(body.name, body.email);
  const res = await this.registerUserService.execute(req);
  return res;
}
```

#### **Struktura (Aplikacja)**
Oto przykład struktury aplikacji dla warstwy Aplikacji:
```text
src/
├── application/
│   ├── services/
│   │   └── register-user.service.ts
│   ├── dto/
│   │   ├── register-user.request.ts
│   │   └── register-user.response.ts
```

---

### **Infrastruktura (Infrastructure)**
> To warstwa, która agreguje całą pracę z zewnętrznymi częściami, takimi jak: bazy danych, zewnętrzne API, system plików, buforowanie, kolejki itp.


#### **Baza Danych (Database)**
> Oto część kodu odpowiedzialna za integrację połączenia z bazą danych. Do realizacji takiego



##### **Mappery (Mappers)** 

> Mappery - to techniczny most między Infrastrukturą <—> Domeną <—> DTO; pomagają nam nie mieszać kodu i nie naruszać `Zasady Pojedynczej Odpowiedzialności (Single Responsibility Principle)`. Głównym powodem używania mapperów jest to, że model domenowy i encja ORM (lub DTO) mogą mieć różną strukturę, obowiązki lub poziom abstrakcji. To właśnie mapper odpowiada za czyste i kontrolowane przekształcanie między tymi warstwami, zachowując czystość architektury i izolację domeny od szczegółów technicznych. Na przykład w modelu domenowym Email jest Obiektem Wartości z walidacją, a w bazie danych jest to zwykły ciąg znaków — mapper hermetyzuje logikę przekształcania między nimi.

**Przykład klasycznego mappera:**

```ts
// user.mapper.ts
import { User } from '../../domain/entities/user.entity';
import { Email } from '../../domain/value-objects/email.vo';
import { Password } from '../../domain/value-objects/password.vo';
import { UserOrmEntity } from '../database/user.orm-entity';

export class UserMapper {
  static async toDomain(entity: UserOrmEntity): Promise<User> {
    return new User(
      entity.userId,
      entity.name,
      entity.surname,
      new Email(entity.email),
      entity.phone,
      Password.fromHashed(entity.password),
      entity.isActive,
      entity.createdAt,
    );
  }

  static toOrm(user: User): UserOrmEntity {
    const raw = user.toPrimitives();
    const orm = new UserOrmEntity();
    orm.userId = raw.id;
    orm.name = raw.name;
    orm.surname = raw.surname;
    orm.email = raw.email;
    orm.phone = raw.phone;
    orm.password = raw.password;
    orm.isActive = raw.isActive;
    orm.createdAt = raw.createdAt;
    return orm;
  }
}
```

#### **Repozytoria (Interfejsy)**
> W warstwie `Infrastructure` możemy opisać interfejsy, które w przyszłości będziemy mogli zaimplementować.

Oto mały przykład, który później zaimplementujemy w warstwie `Infrastructure`:

```ts
// domain/repositories/user.repository.interface.ts
import { User } from "../../domain/entities/user.entity";

export const USER_REPOSITORY = "USER_REPOSITORY";

export interface IUserRepository {
  create(user: User): Promise<User>;
  findById(userId: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findAll(): Promise<User[]>;
  update(userId: string, user: Partial<User>): Promise<User>;
  delete(userId: string): Promise<void>;
}

```

#### **Repozytoria (Implementacje)**
> W tej części implementowane są repozytoria zdefiniowane w domenie; oczywiście nie jest to ścisłe, ale często tak się robi.

```ts
// user.repository.ts
import { Injectable } from '@nestjs/common';
import { Repository } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';
import { IUserRepository } from './user.repository.interface';
import { User } from '../../domain/entities/user.entity';
import { UserOrmEntity } from '../database/user.orm-entity';
import { UserMapper } from './user.mapper';

@Injectable()
export class UserRepository implements IUserRepository {
  constructor(
    @InjectRepository(UserOrmEntity)
    private readonly repository: Repository<UserOrmEntity>
  ) {}

  async create(user: User): Promise<User> {
    const saved = await this.repository.save(UserMapper.toOrm(user));
    return await UserMapper.toDomain(saved);
  }

  async findById(userId: string): Promise<User | null> {
    const entity = await this.repository.findOne({ where: { userId } });
    return entity ? await UserMapper.toDomain(entity) : null;
  }

  async findByEmail(email: string): Promise<User | null> {
    const entity = await this.repository.findOne({ where: { email } });
    return entity ? await UserMapper.toDomain(entity) : null;
  }

  async delete(userId: string): Promise<void> {
    await this.repository.delete({ userId });
  }

  async update(user: User): Promise<User> {
    const updated = await this.repository.save(UserMapper.toOrm(user));
    return await UserMapper.toDomain(updated);
  }
}
```

#### **Struktura (Infrastruktura)**
```text
src/
└── infrastructure/
    ├── config/
    │   ├── database.config.ts        # ustawienia połączenia z bazą danych
    │   ├── cache.config.ts           # ustawienia pamięci podręcznej (Redis)
    │   └── logger.config.ts          # ustawienia logowania (winston/pino)
    │
    ├── database/
    │   ├── entities/                 # schematy ORM
    │       ├── user.orm-entity.ts
    │       ├── order.orm-entity.ts
    │       └── ...  
    │
    ├── repositories/                 # implementacje interfejsów repozytoriów
    │   ├── user.repository.ts        # implementacja IUserRepository przez TypeORM
    │   ├── order.repository.ts       # implementacja IOrderRepository
    │   └── ...
    │
    ├── services/                     # serwisy techniczne (adaptery)
    │   ├── email/
    │   │   ├── email.service.ts      # implementuje IEmailService (SMTP, SendGrid itp.)
    │   │   └── email.module.ts
    │   ├── payment/
    │   │   ├── stripe.service.ts     # implementuje IPaymentGateway
    │   │   └── payment.module.ts
    │   ├── cache/
    │   │   ├── cache.service.ts      # adapter Redis
    │   │   └── cache.module.ts
    │   └── ...
    │
    ├── clients/                      # klienci do zewnętrznych API/serwisów
    │   ├── sms.client.ts             # Twilio, Vonage itp.
    │   ├── geolocation.client.ts     # Google Maps API
    │   └── ...
    │
    ├── messaging/                    # kolejki i zdarzenia
        ├── kafka.producer.ts
        ├── rabbitmq.module.ts
        └── subscribers/              # obsługiwacze zdarzeń przychodzących
            └── order-created.subscriber.ts
```

---

### **Prezentacja (Presentation)**
> Warstwa `Prezentacji` odpowiada za interakcję ze światem zewnętrznym. Realizuje się to za pomocą HTTP, WebSocket, GraphQL itp. Jej zadaniem jest przekształcanie zewnętrznych żądań w DTO, wywoływanie odpowiednich serwisów warstwy aplikacyjnej i zwracanie klientowi odpowiedzi DTO lub odpowiedniego statusu.

#### **Kontrolery / Trasy (Controllers / Routes)**

##### Ogólna idea (czysty Node.js / styl Express):
  - Trasy (Routes) — po prostu rejestracja ścieżek (URL + metoda HTTP) i oprogramowania pośredniczącego (middleware).
  - Kontrolery (Handlers) — funkcje, które:
    1. Przyjmują `req`, `res`, `next` (Express/Koa/Fastify).
    2. Mapują ciało żądania na DTO Aplikacji (Request DTO).
    3. Wywołują potrzebny Serwis Aplikacyjny.
    4. Mapują wynik na DTO Odpowiedzi (Response DTO) i zwracają `res.json(...)` lub `res.status(...).send(...)`.
    5. Obsługują błędy (poprzez `try/catch` + `next(err)` lub przez scentralizowany mechanizm obsługi błędów).

* **Przykład 1:**

```ts
// przykład express
import { Router, Request, Response, NextFunction } from 'express';
import { RegisterUserService } from '../application/services/register-user.service';
import { RegisterUserRequest } from '../application/dto/register-user.request';
import { RegisterUserResponse } from '../application/dto/register-user.response';

const router = Router();
const service = new RegisterUserService(userRepo);

router.post('/users', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const dto = new RegisterUserRequest(req.body.name, req.body.email);
    const result = await service.execute(dto);
    const responseDto = new RegisterUserResponse(result.id, result.email);
    res.status(201).json(responseDto);
  } catch (err) {
    next(err);
  }
});

export default router;
```

* **Przykład 2 (bardziej praktyczny):**

Tutaj zastosowałem podział na kontroler i router, ponieważ zmniejsza to sprzężenie i pozwala dodać więcej logiki w routerze (middleware) bez dodatkowego obciążenia poznawczego.

* **Router:**
```ts
// przykład fastify
export async function userRoutes(fastify: FastifyInstance) {
  fastify.post(
    "/users",
    { preHandler: [authorize([UserRole.ADMIN]), isAdmin()] },
    addUser,
  );

  fastify.get(
    "/users",
    { preHandler: [authorize([UserRole.ADMIN]), isAdmin()] },
    getAllUsers,
  );
  //...
}
```

* **Kontroler:**
```ts
export const addUser = async (request: FastifyRequest, reply: FastifyReply) => {
  try {
    const userDto = CreateUserDTOSchema.parse(request.body);
    const newUser = await UserService.addUser(userDto);
    reply.status(201).send(newUser);
  } catch (error) {
    if (error instanceof z.ZodError) {
      reply
        .code(400)
        .send({ message: "Validation error", errors: error.errors });
    } else {
      logger.error(`Error creating user: ${error}`);
      reply.status(500).send({ message: "Error creating user" });
    }
  }
};

export const getAllUsers = async (_: FastifyRequest, reply: FastifyReply) => {
  try {
    const users = await UserService.getAll();
    reply.status(200).send(users);
  } catch (error) {
    logger.error(`Error getting all users: ${error}`);
    if (error instanceof CustomError) {
      reply.status(error.statusCode).send({ message: error.message });
    } else {
      reply.status(500).send({ message: "Error during get all users" });
    }
  }
};
```

##### Ogólna idea (NestJS):
  - **Kontrolery** — klasy z dekoratorami `@Controller()`, metodami `@Get()`, `@Post()` itp.
  - **Trasy (Routes)** — zostaną wygenerowane automatycznie z tych dekoratorów.
  - **DI (Dependency Injection)** — przez konstruktor(private readonly svc: Service).

* **Przykład:**
```ts
// users.controller.ts
import { Controller, Post, Body } from '@nestjs/common';
import { RegisterUserService } from '../../application/services/register-user.service';
import { RegisterUserDto } from './dto/register-user.dto';

@Controller('users')
export class UsersController {
  constructor(private readonly registerSvc: RegisterUserService) {}

  @Post()
  async register(@Body() body: RegisterUserDto) {
    const result = await this.registerSvc.execute({
      name: body.name,
      email: body.email,
    });
    return { id: result.id, email: result.email };
  }
}
```

#### **DTO (Prezentacja)**
W tej warstwie DTO (Data Transfer Objects) warto podzielić na dwa typy:
* **Request DTO** — struktura, która dokładnie opisuje, jakie dane są przyjmowane z zewnętrznego interfejsu.
  - Używane do walidacji (np. `Joi`, `class-validator`, `zod`).
  - Minimalizują „hardkodowanie” pól w kontrolerach.

* **Response DTO** - struktura, którą odpowiadamy klientowi.
  - Może ukrywać „wrażliwe” pola (`password`, wewnętrzne identyfikatory).
  - Pozwala kontrolować format odpowiedzi.

* **Przykład 1 (styl czystego Node.js):**
```ts
// styl express-fastify
// application/dto/register-user.request.ts
export class RegisterUserRequest {
  constructor(
    public readonly name: string,
    public readonly email: string,
  ) {}
}

// application/dto/register-user.response.ts
export class RegisterUserResponse {
  constructor(
    public readonly id: string,
    public readonly email: string,
  ) {}
}
```

* **Przykład 2 (NestJS):**
```ts
//NestJS
import { IsString, IsEmail } from 'class-validator';

export class RegisterUserDto {
  @IsString() name: string;
  @IsEmail() email: string;
}
```

#### **Struktura (Prezentacja)**
```text
src/
└── interfaces/                   # lub presentation/
    ├── http/                     # HTTP-API
    │   ├── controllers/          # NestJS: klasy kontrolerów
    │   │   └── users.controller.ts
    │   ├── routes/               # Routery Express/Koa/Fastify
    │   │   └── users.routes.ts
    │   ├── dto/                  # DTO HTTP (wejściowe/wyjściowe)
    │   │   ├── register-user.dto.ts
    │   │   └── ...
    │
    ├── graphql/                  # jeśli istnieje GraphQL
    │   ├── resolvers/
    │   └── schemas/
    │
    ├── cli/                      # jeśli istnieje interfejs wiersza poleceń
    │   └── commands/
    │
    └── websocket/                # jeśli istnieje WebSocket/Event
        └── gateways/
```

---

### Wyjątki (Exceptions):

> W tym rozdziale omówimy ważną i nieodłączną część każdej aplikacji — obsługę wyjątków. Jednym z kluczowych aspektów tej pracy jest prawidłowe rzucanie wyjątków: we właściwym miejscu, bez duplikacji i z wyraźnym rozgraniczeniem odpowiedzialności.
>
> Pomaga w tym podejście DDD, gdzie każda warstwa architektury ma swoją strefę odpowiedzialności i pracuje z własnym zestawem wyjątków.

Najpierw musimy określić strefę odpowiedzialności każdej warstwy.

#### Warstwa Domeny (Domain Layer)
W tej warstwie tworzymy podstawowe wyjątki, które są bezpośrednio związane z logiką biznesową. Na przykład, mamy encję `Order` i wywołujemy metodę potwierdzającą zamówienie. Jeśli zamówienie nie zawiera żadnych produktów, zgodnie z regułami biznesowymi, musimy rzucić wyjątek.

Oczywiście, można po prostu napisać:

```ts
throw new Error("Cannot confirm an order with no items.");
```

Ale takie podejście ma istotną wadę: wyjątek nie jest wielokrotnego użytku i trudno go typować lub selektywnie obsługiwać. Jeśli taki błąd powtarza się w różnych miejscach (co jest prawdopodobne), będziemy ciągle powtarzać tę samą konstrukcję `throw new Error(...)`, co prowadzi do duplikacji i naruszenia zasad czystego kodu.
Zamiast tego, bardziej celowe jest stworzenie wyspecjalizowanej klasy wyjątku — na przykład `CannotConfirmEmptyOrderException` — i używanie jej wszędzie tam, gdzie naruszana jest ta reguła biznesowa. Zwiększa to czytelność, ponowne użycie i kontrolę nad błędami.

Oto przykład:

`order.entity.ts`

```ts
import { CannotConfirmEmptyOrderException } from './exceptions/domain.exceptions';

export class Order {
  private confirmed = false;

  constructor(private readonly items: string[]) {}

  confirm(): void {
    if (this.items.length === 0) {
      throw new CannotConfirmEmptyOrderException();
    }
    this.confirmed = true;
  }
}
```

`domain.exceptions.ts`

```ts
export class DomainException extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'DomainException';
  }
}

export class CannotConfirmEmptyOrderException extends DomainException {
  constructor() {
    super('Cannot confirm an order with no items.');
    this.name = 'CannotConfirmEmptyOrderException';
  }
}
```

#### Warstwa Aplikacji (Application Layer)

W tej warstwie realizowane są konkretne przypadki biznesowe (use-case). Kluczowe aspekty:

*   **Mapowanie wyjątków domenowych.** Wszystkie `DomainException` są przechwytywane i przekształcane na wyjątki aplikacyjne `ApplicationException` (z dodaniem kodu błędu, komunikatu i kontekstu).
*   **Walidacja DTO.** Podczas pracy z przychodzącymi DTO wykonywana jest walidacja danych — w przypadku błędów tworzony jest `ValidationException`.
*   **Logowanie.** Każdy krok wykonania przypadku użycia jest śledzony za pomocą logów, aby ułatwić monitorowanie i debugowanie.

W ten sposób warstwa Aplikacji najpierw przechwytuje wyjątki domenowe, dodaje do nich niezbędny kontekst (logika ponawiania, audytu, kody błędów) i zwraca wynik do Warstwy Prezentacji.

Uzupełniając nasz poprzedni przykład z `Order`:

`confirm-order.service.ts`

```ts
import { Injectable, Logger } from '@nestjs/common';
import { OrderRepository } from '../infrastructure/order.repository';
import { CannotConfirmEmptyOrderException } from '../domain/exceptions/domain.exceptions';
import { OrderConfirmationFailedException } from '../application/exceptions/application.exceptions';

@Injectable()
export class ConfirmOrderService {
  private readonly logger = new Logger(ConfirmOrderService.name);

  constructor(private readonly orderRepo: OrderRepository) {}

  async execute(orderId: string): Promise<void> {
    this.logger.log(`Starting confirmation for Order ${orderId}`);

    let order;
    try {
      order = await this.orderRepo.findById(orderId);
      this.logger.debug(`Order ${orderId} fetched successfully`);
    } catch (err) {
      this.logger.error(
        `Failed to fetch Order ${orderId}`,
        err instanceof Error ? err.stack : String(err),
      );
      // tutaj próbujemy rejestrować audyt lub ponowienia
      throw new OrderConfirmationFailedException(
        'Could not retrieve order',
        'order.fetch_failed',
        err as Error,
      );
    }

    try {
      order.confirm();
      await this.orderRepo.save(order);
      this.logger.log(`Order ${orderId} confirmed and saved`);
    } catch (err) {
      if (err instanceof CannotConfirmEmptyOrderException) {
        this.logger.warn(
          `Business rule violation on Order ${orderId}: ${err.message}`,
        );
        throw new OrderConfirmationFailedException(
          err.message,
          'order.empty',
          err,
        );
      }

      this.logger.error(
        `Unexpected error on Order ${orderId} confirmation`,
        err instanceof Error ? err.stack : String(err),
      );
      throw err; // nieoczekiwane błędy idą dalej
    }
  }
}
```

`application.exceptions.ts`

```ts
export class ApplicationException extends Error {
  constructor(message: string, public readonly code: string, public readonly cause?: Error) {
    super(message);
    this.name = 'ApplicationException';
    if (cause) this.stack = cause.stack;
  }
}

export class OrderConfirmationFailedException extends ApplicationException {
  constructor(message: string, code: string, cause?: Error) {
    super(message, code, cause);
    this.name = 'OrderConfirmationFailedException';
  }
}
```

#### Warstwa Infrastruktury (Infrastructure Layer)

W tej warstwie realizowane są integracje z usługami zewnętrznymi (bazy danych, zewnętrzne API, cache, kolejki itp.) i, odpowiednio, pracujemy z błędami, które te
zewnętrzne usługi nam zwracają. Na tym poziomie typowe są: `SqlException`, `IOException`, `HttpRequestException` itd.

Aby izolować inne warstwy od szczegółów implementacji i uniknąć „wycieku” informacji technicznych, my:

1.  **Przechwytujemy niskopoziomowe wyjątki** od sterowników i klientów usług zewnętrznych.
2.  **Przepakowujemy** je w zrozumiałe dla systemu abstrakcje — własne, niestandardowe błędy, takie jak `StorageUnavailableException`, `CacheAccessException`, `MessageQueueException` itd.
3.  **Logujemy** każdy błąd z kontekstem (nazwa operacji, parametry zapytania, sygnatura czasowa) do centralnego systemu monitorowania lub serwera audytu.

```ts
try {
  await this.db.query(/* … */);
} catch (err) {
  this.logger.error('DB query failed', err.stack);
  throw new StorageUnavailableException(err);
}
```

**Dlaczego to ważne?**

*   **Izolacja warstw.** Kod domenowy i aplikacyjny nie widzi i nie zależy od specyfiki sterownika SQL ani klienta HTTP.
*   **Czystsze śledzenie błędów.** Zawsze mają ten sam format, zawierają własne kody i komunikaty, a także „cause” (oryginalny błąd) do dalszej analizy.
*   **Możliwość ponawiania i circuit-breaker.** W przypadku natychmiastowych awarii łatwo zastosować strategie ponawiania lub ograniczenia obciążenia usługi.

Przykład implementacji:

`storage.exceptions.ts`

```ts
export class StorageUnavailableException extends Error {
  constructor(cause?: unknown) {
    super('Storage service is unavailable');
    this.name = 'StorageUnavailableException';
    if (cause instanceof Error) {
      // zachowujemy oryginalny stos do diagnostyki
      this.stack = cause.stack;
    }
  }
}
```

`not-found.exceptions.ts`

```ts
export class OrderNotFoundException extends Error {
  constructor(orderId: string) {
    super(`Order with id=${orderId} not found`);
    this.name = 'OrderNotFoundException';
  }
}
```

`order.repository.ts`

```ts
import { Injectable, Logger } from '@nestjs/common';
import { Repository, EntityNotFoundError } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';

import { OrderEntity } from './order.entity';
import { Order } from '../domain/order';
import { OrderMapper } from './order.mapper';
import { StorageUnavailableException } from './exceptions/storage.exceptions';
import { OrderNotFoundException } from './exceptions/not-found.exceptions';

@Injectable()
export class OrderRepository {
  private readonly logger = new Logger(OrderRepository.name);

  constructor(
    @InjectRepository(OrderEntity)
    private readonly repo: Repository<OrderEntity>,
    private readonly mapper: OrderMapper,
  ) {}

  async findById(id: string): Promise<Order> {
    try {
      this.logger.debug(`Fetching Order ${id} from database`);
      const entity = await this.repo.findOneOrFail({ where: { id } });
      this.logger.debug(`Order ${id} found, mapping to domain`);
      return this.mapper.toDomain(entity);
    } catch (e) {
      // jeśli rekord nie został znaleziony — rzucamy wyjątek
      if (e instanceof EntityNotFoundError) {
        this.logger.warn(`Order ${id} not found`);
        throw new OrderNotFoundException(id);
      }
      // wszystkie inne błędy — techniczne, przepakowujemy
      this.logger.error(
        `Error fetching Order ${id}`,
        e instanceof Error ? e.stack : String(e),
      );
      throw new StorageUnavailableException(e);
    }
  }

  async save(order: Order): Promise<void> {
    try {
      this.logger.debug(`Saving Order ${order.id} to database`);
      const entity = this.mapper.toEntity(order);
      await this.repo.save(entity);
      this.logger.log(`Order ${order.id} saved successfully`);
    } catch (e) {
      this.logger.error(
        `Error saving Order ${order.id}`,
        e instanceof Error ? e.stack : String(e),
      );
      throw new StorageUnavailableException(e);
    }
  }
}
```

#### Warstwa Prezentacji (Presentation Layer)

Na poziomie Warstwy Prezentacji przekształcamy wyniki wykonania przypadku użycia (lub rzucone wyjątki) w poprawne odpowiedzi HTTP (i nie tylko). Oto kilka kluczowych punktów:
1. Przechwytywanie wyników
Kontroler otrzymuje albo Result z serwisu, albo przekazane wyjątki, i mapuje je na standardowe wyjątki NestJS:

    *   `BadRequestException` (HTTP 400) dla błędów walidacji lub reguł biznesowych.
    *   `NotFoundException` (HTTP 404) jeśli zasób nie został znaleziony.
    *   `ServiceUnavailableException` (HTTP 503) dla niedostępności usług zewnętrznych.
    *   `InternalServerErrorException` (HTTP 500) dla wszystkich innych.

2. Globalny filtr:
Dla wszystkich nieprzechwyconych błędów rejestrujemy `AllExceptionsFilter`, który tworzy jednolity format odpowiedzi i loguje błędy.

Oto przykład implementacji:

`order-exceptions.filter.ts`

```ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  BadRequestException,
  NotFoundException,
  ServiceUnavailableException,
  Injectable,
} from '@nestjs/common';
import { Response } from 'express';
import { OrderConfirmationFailedException } from '../../application/exceptions/application.exceptions';
import { OrderNotFoundException } from '../../infrastructure/exceptions/not-found.exceptions';

@Catch(OrderConfirmationFailedException, OrderNotFoundException)
@Injectable()
export class OrderExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx     = host.switchToHttp();
    const response: Response = ctx.getResponse<Response>();

    if (exception instanceof OrderConfirmationFailedException) {
      switch (exception.code) {
        case 'order.empty':
          throw new BadRequestException(exception.message);
        case 'order.fetch_failed':
          throw new NotFoundException(exception.message);
        case 'storage.unavailable':
          throw new ServiceUnavailableException(exception.message);
        default:
          throw new HttpException(exception.message, 500);
      }
    }

    if (exception instanceof OrderNotFoundException) {
      throw new NotFoundException(exception.message);
    }

    throw exception;
  }
}
```

`order.controller.ts`

```ts
import { Controller, Post, Param, UseFilters, Logger } from '@nestjs/common';
import { ConfirmOrderService } from '../../application/confirm-order.service';
import { OrderExceptionsFilter } from './order-exceptions.filter';

@Controller('orders')
@UseFilters(OrderExceptionsFilter)
export class OrderController {
  private readonly logger = new Logger(OrderController.name);

  constructor(private readonly confirmOrder: ConfirmOrderService) {}

  @Post(':id/confirm')
  async confirm(@Param('id') id: string): Promise<{ message: string }> {
    this.logger.log(`HTTP POST /orders/${id}/confirm`);
    await this.confirmOrder.execute(id);
    return { message: 'Order confirmed successfully.' };
  }
}
```

Wniosek:

Faktycznie nasza praca z wyjątkami dzieli się na dwa etapy: Przechwytywanie wyjątków → Wzbogacanie kontekstu, gdzie

*   **Przechwytywanie wyjątków:** Otaczamy kod przypadku użycia w `try…catch` i przechwytujemy zarówno oczekiwane wyjątki domenowe lub infrastrukturalne, jak i wszelkie nieoczekiwane błędy.
*   **Wzbogacanie kontekstu:** Po przechwyceniu błędu zapisujemy kluczowe dane (ID użytkownika/sesji, parametry wejściowe, stan obiektów, sygnatury czasowe, ślad stosu), aby uprościć debugowanie.

Krótkie podsumowanie w formie tabeli:

| **Warstwa (Layer)** | **Typowe wyjątki (Exceptions)**                                                               | **Przeznaczenie**                                                           |
| ------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **Domena (Domain)** | `DomainException`, `BusinessRuleViolationException`, `InvalidStateException`                  | Naruszenie reguł biznesowych, niezmienników lub stanu encji               |
| **Aplikacja (Application)** | `ValidationException`, `UseCaseError`, `DomainException`                                | Nieprawidłowe dane wejściowe, obsługa wyjątków domenowych, agregacja błędów |
| **Infrastruktura (Infrastructure)** | `SqlException`, `IOException`, `HttpRequestException`, `StorageUnavailableException`  | Błędy techniczne dostępu do bazy danych, plików, sieci, API                 |
| **Prezentacja (Presentation)** | `BadRequestException`, `NotFoundException`, `InternalServerErrorException`, `ExceptionHandler` | Mapowanie błędów na odpowiedzi HTTP lub komunikaty dla użytkownika         |

---

### Podsumowanie:
Podsumowując, warto podkreślić, że w kontekście wielu stylów architektonicznych (`Clean Architecture`, `Hexagonal`, `Onion`, `DDD`) istnieje `„Zasada Zależności” (Dependency Rule)`, która formułowana jest następująco:
> **Kod wewnątrz jednej warstwy nie może zależeć od kodu warstwy zewnętrznej.** Innymi słowy, strzałki zależności zawsze kierują się do wewnątrz, od mniej stabilnych/bardziej aplikacyjnych modułów do bardziej stabilnych/bardziej abstrakcyjnych.

Jeśli przedstawimy to obrazowo, wyjdzie coś takiego:
`Prezentacja → Aplikacja → Domena ← Infrastruktura`.

![Flow Schema](../images/flow-schema.png)

A oto krótka tabelka podsumowująca wszystkie rozdziały opisane w artykule:
| Warstwa (Layer)      | Główne obowiązki                                                                                                | Kluczowe komponenty                                                                                                                              |
|------------------|-------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| **Domena (Domain)**       | Definicja i hermetyzacja logiki biznesowej, reguł dziedziny, niezmienników i stanu.                          | Encje (Entities), Obiekty Wartości (Value Objects), Agregaty (Aggregates), Serwisy Domenowe (Domain Services), Interfejsy Repozytoriów.        |
| **Aplikacja (Application)**  | Orkiestracja scenariuszy użycia (use cases), koordynacja interakcji między obiektami domenowymi a infrastrukturą. | Serwisy Aplikacji (Application Services), Polecenia (Commands), Zapytania (Queries), DTO Aplikacji (Application DTOs dla danych wejściowych/wyjściowych serwisów). |
| **Infrastruktura (Infrastructure)**| Implementacja interakcji z systemami zewnętrznymi (bazy danych, zewnętrzne API, system plików, kolejki komunikatów, pamięć podręczna).  | Implementacje Repozytoriów, Encje ORM, Mappery (Mappers), Adaptery/Bramy (Adapters/Gateways) do zewnętrznych serwisów, konfiguracja, loggery.     |
| **Prezentacja (Presentation)** | Wzajemne oddziaływanie z użytkownikiem lub innymi systemami klienckimi (HTTP API, WebSocket, GraphQL, CLI). | Kontrolery/Routery (Controllers/Routes), DTO Prezentacji (Presentation DTOs dla żądań/odpowiedzi), Obsługiwacze Zdarzeń, UI (jeśli istnieje).   |

Przestrzeganie takiego podziału na warstwy i jasnego zdefiniowania ich obowiązków pomaga budować systemy, które są:
- **Testowalne:** Logika biznesowa w `Domain` jest izolowana i może być testowana bez zależności od UI czy bazy danych.
- **Elastyczne:** Łatwiej wymieniać technologie w `Infrastructure` (np. bazę danych) lub sposoby interakcji w Prezentacji bez znaczącego wpływu na inne warstwy.
- **Zrozumiałe:** Przejrzysta struktura ułatwia zrozumienie kodu i wprowadzenie nowych programistów do projektu.
- **Skalowalne i Łatwe w Utrzymaniu:** Logiczny podział upraszcza rozszerzanie funkcjonalności i długoterminowe utrzymanie systemu.

---

### Odnośniki

* (Eric Evans - "Domain-Driven Design Tackling Complexity in the Heart of Software")[https://fabiofumarola.github.io/nosql/readingMaterial/Evans03.pdf]
* (Marco Lenzo - "Domain-Driven Aggregates Explained | Why you should use them")[https://youtu.be/SvnsOX4oVVo?si=cGBi1kU4jKDSb7Ar]
* (Roman Dykyi - "Error handling and strategies")[https://medium.com/@dykyi.roman/error-handling-and-strategies-a55b5a285b6b]