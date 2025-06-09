
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
    *   1.3 [Repozytoria (Interfejsy)](#repozytoria-interfejsy)
    *   1.4 [Agregaty (Aggregates)](#agregaty-aggregates)
    *   1.5 [Fabryki (Factories)](#fabryki-factories)
2.  [**Aplikacja (Application)**](#aplikacja-application)
    *   2.1 [Serwisy (Services)](#serwisy-services)
    *   2.2 [DTO Aplikacji (Application DTOs)](#dto-aplikacji-application-dtos)
    *   2.3 [Struktura (Aplikacja)](#struktura-aplikacja)
3.  [**Infrastruktura (Infrastructure)**](#infrastruktura-infrastructure)
    *   3.1 [Baza Danych (Database)](#baza-danych-database)
        *   3.1.1 [Mappery (Mappers)](#mappery-mappers)
    *   3.2 [Repozytoria (Implementacje)](#repozytoria-implementacje)
    *   3.3 [Struktura (Infrastruktura)](#struktura-infrastruktura)
4.  [**Prezentacja (Presentation)**](#prezentacja-presentation)
    *   4.1 [Kontrolery / Trasy (Controllers / Routes)](#kontrolery--trasy-controllers--routes)
    *   4.2 [DTO (Prezentacja)](#dto-prezentacja)
    *   4.3 [Struktura (Prezentacja)](#struktura-prezentacja)
5.  [**Podsumowanie**](#podsumowanie)
6.  [**Odnośniki**](#odnośniki)

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

> Następną częścią warstwy domenowej są Obiekty Wartości (Value Objects) - to faktycznie część, która implementuje dostęp do bardziej złożonych pól: Hasło, Email, NumerTelefonu, gdzie musimy przeprowadzać pewne sprawdzenia lub walidację zgodnie z zdefiniowanymi przez nas regułami biznesowymi.

Główne właściwości, które określają odrębność Obiektów Wartości:
 * hermetyzuje wartość
 * gwarantuje ich poprawność zgodnie z regułami biznesowymi
 * nie ma tożsamości (w przeciwieństwie do Encji)
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

#### **Repozytoria (Interfejsy)**
> W warstwie `Domain` możemy opisać interfejsy, które w przyszłości będziemy mogli zaimplementować w warstwie `Infrastructure`.

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

Tutaj zastosowałem podział na kontroler i router, ponieważ zmniejsza to sprzężenie i pozwala dodać więcej logiki do routera (middleware) bez dodatkowego obciążenia poznawczego.

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
| **Prezentacja (Presentation)** | Interakcja z użytkownikiem lub innymi systemami klienckimi (HTTP API, WebSocket, GraphQL, CLI).                     | Kontrolery/Routery (Controllers/Routes), DTO Prezentacji (Presentation DTOs dla żądań/odpowiedzi), Obsługiwacze Zdarzeń, UI (jeśli istnieje).   |

Przestrzeganie takiego podziału na warstwy i jasnego zdefiniowania ich obowiązków pomaga budować systemy, które są:
- **Testowalne:** Logika biznesowa w `Domain` jest izolowana i może być testowana bez zależności od UI czy bazy danych.
- **Elastyczne:** Łatwiej wymieniać technologie w `Infrastructure` (np. bazę danych) lub sposoby interakcji w Prezentacji bez znaczącego wpływu na inne warstwy.
- **Zrozumiałe:** Przejrzysta struktura ułatwia zrozumienie kodu i wprowadzenie nowych programistów do projektu.
- **Skalowalne i Łatwe w Utrzymaniu:** Logiczny podział upraszcza rozszerzanie funkcjonalności i długoterminowe utrzymanie systemu.

---

### Odnośniki

* (Eric Evans - "Domain-Driven Design Tackling Complexity in the Heart of Software")[https://fabiofumarola.github.io/nosql/readingMaterial/Evans03.pdf]
* (Marco Lenzo - "Domain-Driven Aggregates Explained | Why you should use them")[https://youtu.be/SvnsOX4oVVo?si=cGBi1kU4jKDSb7Ar]