# Практическая работа №8: ООП в Dart — решения
> Каждое решение — отдельный фрагмент. Имена классов в разных заданиях повторяются, поэтому запускай каждое в своём `.dart`-файле (или DartPad).
---
## Тема 1. Классы, инкапсуляция, геттеры/сеттеры
### 1. Thermostat
```dart
class Thermostat {
double _temperatureCelsius = 20;
double get celsius => _temperatureCelsius;
double get fahrenheit => _temperatureCelsius * 9 / 5 + 32;
set celsius(double value) {
if (value < -50 || value > 100) {
throw RangeError("Температура должна быть в диапазоне от -50 до +100 °C");
}
_temperatureCelsius = value;
}
}
void main() {
final t = Thermostat();
t.celsius = 25;
print("${t.celsius} °C = ${t.fahrenheit} °F");
try {
t.celsius = 150;
} catch (e) {
print(e);
}
}
```
### 2. Pedometer
```dart
class Pedometer {
int _steps = 0;
int get steps => _steps;
void addSteps(int count) {
if (count <= 0) {
throw ArgumentError("Шаги можно только увеличивать");
}
_steps += count;
}
}
void main() {
final p = Pedometer();
p.addSteps(100);
p.addSteps(50);
print(p.steps);
}
```
### 3. Book
```dart
class Book {
final String title;
final int pages;
Book(this.title, this.pages);
double get estimatedReadingHours => pages * 2 / 60;
}
void main() {
final b = Book("Война и мир", 1225);
print("${b.title}: ${b.estimatedReadingHours.toStringAsFixed(1)} ч");
}
```
### 4. UserCredentials
```dart
class UserCredentials {
String _passwordHash = "";
set password(String raw) {
if (raw.length < 8) {
throw ArgumentError("Пароль должен содержать не менее 8 символов");
}
_passwordHash = "hash_${raw.hashCode}"; // псевдо-хэш
}
bool check(String raw) => _passwordHash == "hash_${raw.hashCode}";
}
void main() {
final u = UserCredentials();
u.password = "superSecret1";
print(u.check("superSecret1"));
print(u.check("wrong"));
}
```
### 5. DeviceSession
```dart
class DeviceSession {
static int activeSessionsCount = 0;
final String deviceName;
DeviceSession(this.deviceName) {
activeSessionsCount++;
}
}
void main() {
DeviceSession("Phone");
DeviceSession("Tablet");
DeviceSession("Laptop");
print(DeviceSession.activeSessionsCount); // 3
}
```
---
## Тема 2. Конструкторы
### 1. AppColor
```dart
class AppColor {
final int r, g, b;
final double opacity;
AppColor.fromRGBO(this.r, this.g, this.b, this.opacity);
factory AppColor.fromHex(String hex) {
hex = hex.replaceAll("#", "");
if (hex.length != 6) {
throw FormatException("Ожидается формат RRGGBB");
}
return AppColor.fromRGBO(
int.parse(hex.substring(0, 2), radix: 16),
int.parse(hex.substring(2, 4), radix: 16),
int.parse(hex.substring(4, 6), radix: 16),
1.0,
);
}
@override
String toString() => "rgba($r, $g, $b, $opacity)";
}
void main() {
print(AppColor.fromHex("#FF8800"));
print(AppColor.fromRGBO(10, 20, 30, 0.5));
}
```
### 2. Point2D
```dart
class Point2D {
final double x, y;
const Point2D(this.x, this.y);
}
void main() {
const p1 = Point2D(1, 2);
const p2 = Point2D(1, 2);
print(identical(p1, p2)); // true
}
```
### 3. Rectangle
```dart
class Rectangle {
final double width, height;
Rectangle(this.width, this.height) : assert(width > 0 && height > 0, "Стороны должны быть положительными");
double get area => width * height;
}
void main() {
print(Rectangle(3, 4).area);
// Rectangle(-1, 4); // AssertionError (запуск с включёнными assert)
}
```
### 4. ApiClient (синглтон)
```dart
class ApiClient {
static final ApiClient _instance = ApiClient._internal();
ApiClient._internal();
factory ApiClient() => _instance;
}
void main() {
print(identical(ApiClient(), ApiClient())); // true
}
```
### 5. Circle (redirecting)
```dart
class Circle {
final double radius;
Circle(this.radius);
Circle.unit() : this(1.0);
}
void main() {
print(Circle.unit().radius); // 1.0
}
```
---
## Тема 3. Наследование
### 1. Банковские карты
```dart
class Card {
final String number;
double balance;
Card(this.number, this.balance);
bool pay(double amount) {
if (amount > balance) return false;
balance -= amount;
return true;
}
}
class DebitCard extends Card {
DebitCard(super.number, super.balance);
}
class CreditCard extends Card {
final double overdraftLimit;
CreditCard(super.number, super.balance, this.overdraftLimit);
@override
bool pay(double amount) {
if (amount > balance + overdraftLimit) return false;
balance -= amount;
return true;
}
}
void main() {
final d = DebitCard("1111", 100);
final c = CreditCard("2222", 100, 500);
print(d.pay(300)); // false
print(c.pay(300)); // true
print(c.balance); // -200
}
```
### 2. Полиморфный список
```dart
abstract class BaseScreen {
void render();
}
class HomeScreen extends BaseScreen {
@override
void render() => print("Главный экран");
}
class ProfileScreen extends BaseScreen {
@override
void render() => print("Профиль");
}
class SettingsScreen extends BaseScreen {
@override
void render() => print("Настройки");
}
void main() {
final List<BaseScreen> screens = [HomeScreen(), ProfileScreen(), SettingsScreen()];
for (final s in screens) {
s.render();
}
}
```
### 3. super-параметры
```dart
class Base {
final int id;
final String? title;
Base({required this.id, this.title});
}
class SubClass extends Base {
SubClass({required super.id, super.title});
}
void main() {
final s = SubClass(id: 1, title: "Тест");
print("${s.id} ${s.title}");
}
```
### 4. covariant
```dart
class Animal {}
class Cat extends Animal {}
class Food {
void eat(Animal a) => print("Ест животное");
}
class CatFood extends Food {
@override
void eat(covariant Cat c) => print("Ест кошка");
}
void main() {
CatFood().eat(Cat());
// Food f = CatFood(); f.eat(Animal()); // TypeError во время выполнения
}
```
### 5. @nonVirtual
```dart
import 'package:meta/meta.dart'; // во Flutter: package:flutter/foundation.dart
class BaseApp {
@nonVirtual
bool checkLicense(String key) => key == "VALID-KEY";
}
class MyApp extends BaseApp {
// @override bool checkLicense(String key) => true; // предупреждение анализатора
}
void main() {
print(MyApp().checkLicense("VALID-KEY"));
}
```
---
## Тема 4. Абстрактные классы, интерфейсы, sealed
### 1. StorageService
```dart
abstract class StorageService {
void save(String key, String val);
String? read(String key);
void delete(String key);
}
class MemoryStorageService implements StorageService {
final Map<String, String> _data = {};
@override
void save(String key, String val) => _data[key] = val;
@override
String? read(String key) => _data[key];
@override
void delete(String key) => _data.remove(key);
}
void main() {
final s = MemoryStorageService();
s.save("token", "abc");
print(s.read("token"));
s.delete("token");
print(s.read("token")); // null
}
```
### 2. SmartTv
```dart
abstract class Displayable {
void showImage(String url);
}
abstract class NetworkConnectable {
void connect(String ssid);
}
class SmartTv implements Displayable, NetworkConnectable {
@override
void showImage(String url) => print("Показ: $url");
@override
void connect(String ssid) => print("Подключено к $ssid");
}
void main() {
final tv = SmartTv();
tv.connect("HomeWiFi");
tv.showImage("poster.png");
}
```
### 3. AuthState
```dart
class User {
final String name;
User(this.name);
}
sealed class AuthState {}
class Unauthenticated extends AuthState {}
class Authenticating extends AuthState {}
class Authenticated extends AuthState {
final User user;
Authenticated(this.user);
}
class AuthFailed extends AuthState {
final String reason;
AuthFailed(this.reason);
}
String describe(AuthState s) => switch (s) {
Unauthenticated() => "Войдите в аккаунт",
Authenticating() => "Проверка...",
Authenticated(:final user) => "Привет, ${user.name}",
AuthFailed(:final reason) => "Ошибка: $reason",
};
void main() {
print(describe(Authenticated(User("Анна"))));
print(describe(AuthFailed("Неверный пароль")));
}
```
### 4. base class
```dart
// engine.dart
base class Engine {
void start() => print("Двигатель запущен");
}
// В другой библиотеке:
// base class SportEngine extends Engine {} // можно
// class Fake implements Engine {} // ОШИБКА: implements запрещён
```
### 5. interface class
```dart
// contract.dart
interface class PaymentContract {
void pay(double amount) => print("Оплата $amount");
}
// В другой библиотеке:
// class MyPay implements PaymentContract { @override void pay(double a) {} } // можно
// class MyPay2 extends PaymentContract {} // ОШИБКА: extends запрещён
```
---
## Тема 5. Миксины
### 1. JsonPrintable
```dart
import 'dart:convert';
mixin JsonPrintable {
Map<String, dynamic> toJson();
void printAsJson() => print(jsonEncode(toJson()));
}
class Person with JsonPrintable {
final String name;
final int age;
Person(this.name, this.age);
@override
Map<String, dynamic> toJson() => {"name": name, "age": age};
}
void main() => Person("Иван", 20).printAsJson();
```
### 2. AutoDisposer
```dart
import 'package:flutter/foundation.dart';
mixin AutoDisposer on ChangeNotifier {
bool isDisposed = false;
@override
void notifyListeners() {
if (isDisposed) return;
super.notifyListeners();
}
@override
void dispose() {
isDisposed = true;
super.dispose();
}
}
class CounterModel extends ChangeNotifier with AutoDisposer {
int value = 0;
void inc() {
value++;
notifyListeners();
}
}
```
### 3. AnalyticsTracker
```dart
mixin AnalyticsTracker {
void trackEvent(String eventName) => print("[Analytics] $eventName");
}
class CustomButton with AnalyticsTracker {
final String label;
CustomButton(this.label);
void onTap() {
trackEvent("tap_$label");
print("Нажата кнопка $label");
}
}
void main() => CustomButton("buy").onTap();
```
### 4. Линеаризация миксинов
```dart
mixin A {
void execute() => print("A");
}
mixin B {
void execute() => print("B");
}
class Worker with A, B {}
void main() {
Worker().execute(); // B — побеждает последний (правый) миксин
}
```
### 5. MemoizationMixin
```dart
mixin MemoizationMixin {
final Map<String, num> _cache = {};
num memoize(String key, num Function() compute) => _cache.putIfAbsent(key, () {
print("Вычисляем для $key");
return compute();
});
}
class Calc with MemoizationMixin {
num fib(int n) => memoize("fib_$n", () => n < 2 ? n : fib(n - 1) + fib(n - 2));
}
void main() {
final c = Calc();
print(c.fib(30));
print(c.fib(30)); // из кэша
}
```
---
## 50 смешанных заданий
### Блок 1: Модели данных, DTO и конструкторы
### 1. Immutable Product
```dart
class Product {
final String id, name;
final double price, discountPercent;
const Product({required this.id, required this.name, required this.price, this.discountPercent = 0})
: assert(price >= 0, "Цена не может быть отрицательной"),
assert(discountPercent >= 0 && discountPercent <= 100);
double get discountedPrice => price * (1 - discountPercent / 100);
}
void main() {
const p = Product(id: "1", name: "Кроссовки", price: 5000, discountPercent: 20);
print(p.discountedPrice); // 4000.0
}
```
### 2. User.fromJson с безопасными дефолтами
```dart
class User {
final int id;
final String name, email;
User({required this.id, required this.name, required this.email});
factory User.fromJson(Map<String, dynamic> json) => User(
id: (json["id"] is int) ? json["id"] as int : int.tryParse("${json["id"]}") ?? 0,
name: json["name"] is String ? json["name"] as String : "Без имени",
email: json["email"] is String ? json["email"] as String : "",
);
}
void main() {
final u = User.fromJson({"id": "42", "name": null});
print("${u.id} ${u.name} '${u.email}'");
}
```
### 3. AppLocale (multiton)
```dart
class AppLocale {
final String code;
static final Map<String, AppLocale> _instances = {};
AppLocale._(this.code);
factory AppLocale(String code) => _instances.putIfAbsent(code, () => AppLocale._(code));
}
void main() {
print(identical(AppLocale("ru"), AppLocale("ru"))); // true
print(identical(AppLocale("ru"), AppLocale("en"))); // false
}
```
### 4. GeoPoint.fromString
```dart
class GeoPoint {
final double lat, lon;
GeoPoint(this.lat, this.lon);
factory GeoPoint.fromString(String s) {
final parts = s.split(",");
if (parts.length != 2) throw FormatException("Ожидается 'lat, lon'");
return GeoPoint(double.parse(parts[0].trim()), double.parse(parts[1].trim()));
}
@override
String toString() => "($lat, $lon)";
}
void main() => print(GeoPoint.fromString("55.751244, 37.618423"));
```
### 5. ContactCard.fromDatabaseRecord
```dart
class ContactCard {
final int id;
final String name;
final String? phone;
ContactCard(this.id, this.name, this.phone);
factory ContactCard.fromDatabaseRecord(Map<String, Object?> row) => ContactCard(
row["id"] as int,
row["name"] as String,
row["phone"] as String?,
);
}
void main() {
final c = ContactCard.fromDatabaseRecord({"id": 1, "name": "Олег", "phone": null});
print("${c.id} ${c.name} ${c.phone}");
}
```
### 6. copyWith
```dart
class UserModel {
final String name;
final String avatarUrl;
const UserModel({required this.name, required this.avatarUrl});
UserModel copyWith({String? name, String? avatarUrl}) => UserModel(
name: name ?? this.name,
avatarUrl: avatarUrl ?? this.avatarUrl,
);
}
void main() {
const u = UserModel(name: "Аня", avatarUrl: "a.png");
final u2 = u.copyWith(name: "Анна");
print("${u2.name} ${u2.avatarUrl}");
}
```
### 7. DateRange
```dart
class DateRange {
final DateTime start, end;
DateRange(this.start, this.end) : assert(!end.isBefore(start), "Дата окончания раньше даты начала");
Duration get length => end.difference(start);
}
void main() {
print(DateRange(DateTime(2026, 1, 1), DateTime(2026, 1, 10)).length.inDays); // 9
// DateRange(DateTime(2026, 2, 1), DateTime(2026, 1, 1)); // AssertionError
}
```
### 8. TransactionDto
```dart
class TransactionDto {
final String type;
final double amount;
final String? from, to;
TransactionDto._(this.type, this.amount, {this.from, this.to});
TransactionDto.deposit(double amount, String to) : this._("deposit", amount, to: to);
TransactionDto.withdrawal(double amount, String from) : this._("withdrawal", amount, from: from);
TransactionDto.transfer(double amount, String from, String to) : this._("transfer", amount, from: from, to: to);
@override
String toString() => "$type: $amount ($from -> $to)";
}
void main() {
print(TransactionDto.deposit(100, "ACC1"));
print(TransactionDto.transfer(50, "ACC1", "ACC2"));
}
```
### 9. EnvConfig
```dart
class EnvConfig {
final String apiUrl, mode;
final bool debug;
const EnvConfig._(this.apiUrl, this.mode, this.debug);
factory EnvConfig.fromEnvironment() => const EnvConfig._(
String.fromEnvironment("API_URL", defaultValue: "https://api.example.com"),
String.fromEnvironment("MODE", defaultValue: "dev"),
bool.fromEnvironment("DEBUG", defaultValue: true),
);
}
// dart run --define=API_URL=https://prod.com --define=MODE=prod file.dart
void main() => print(EnvConfig.fromEnvironment().apiUrl);
```
### 10. CartItem ==, hashCode
```dart
class CartItem {
final int productId;
final String selectedSize;
final int quantity;
CartItem(this.productId, this.selectedSize, this.quantity);
@override
bool operator ==(Object other) =>
identical(this, other) ||
other is CartItem && other.productId == productId && other.selectedSize == selectedSize;
@override
int get hashCode => Object.hash(productId, selectedSize);
}
void main() {
print(CartItem(1, "M", 1) == CartItem(1, "M", 5)); // true
print({CartItem(1, "M", 1), CartItem(1, "M", 2)}.length); // 1
}
```
### Блок 2: Наследование и иерархия UI-компонентов
### 11. Уведомления
```dart
class InAppNotification {
final String title;
final Duration timeout;
final String icon;
final int color;
InAppNotification(this.title, {this.timeout = const Duration(seconds: 3), this.icon = "info", this.color = 0xFF2196F3});
void show() => print("[$icon] $title (цвет: ${color.toRadixString(16)}, ${timeout.inSeconds}c)");
}
class SuccessNotification extends InAppNotification {
SuccessNotification(super.title) : super(icon: "check", color: 0xFF4CAF50);
}
class WarningNotification extends InAppNotification {
WarningNotification(super.title) : super(icon: "warning", color: 0xFFFFC107, timeout: const Duration(seconds: 5));
}
class CriticalAlertNotification extends InAppNotification {
CriticalAlertNotification(super.title) : super(icon: "error", color: 0xFFF44336, timeout: const Duration(seconds: 10));
}
void main() {
SuccessNotification("Сохранено").show();
CriticalAlertNotification("Нет сети").show();
}
```
### 12. Диалоги
```dart
abstract class BaseDialog {
String get title;
void buildBody();
void show() {
print("=== $title ===");
buildBody();
print("--- подвал диалога ---");
}
}
class ConfirmActionDialog extends BaseDialog {
@override
String get title => "Подтверждение";
@override
void buildBody() => print("Вы уверены? [Подтвердить] [Отмена]");
}
void main() => ConfirmActionDialog().show();
```
### 13. Медиаплееры
```dart
abstract class MediaPlayer {
bool isPlaying = false;
int position = 0;
void play() {
isPlaying = true;
print("${runtimeType}: play");
}
void pause() {
isPlaying = false;
print("${runtimeType}: pause");
}
void seek(int seconds) {
position = seconds;
print("${runtimeType}: seek $seconds");
}
}
class AudioPlayer extends MediaPlayer {}
class HlsVideoPlayer extends MediaPlayer {
@override
void play() {
print("Загрузка HLS-плейлиста...");
super.play();
}
}
void main() {
AudioPlayer().play();
HlsVideoPlayer().play();
}
```
### 14. Сенсоры
```dart
abstract class HardwareSensor {
String get name;
String read();
}
class AccelerometerSensor extends HardwareSensor {
final double x, y, z;
AccelerometerSensor(this.x, this.y, this.z);
@override
String get name => "Акселерометр";
@override
String read() => "x=$x y=$y z=$z";
}
class BarometerSensor extends HardwareSensor {
final double pressureHpa;
BarometerSensor(this.pressureHpa);
@override
String get name => "Барометр";
@override
String read() => "$pressureHpa гПа";
}
void main() {
for (final s in [AccelerometerSensor(0, 0, 9.8), BarometerSensor(1013.2)]) {
print("${s.name}: ${s.read()}");
}
}
```
### 15. Тарифы подписки
```dart
abstract class SubscriptionPlan {
final double basePrice;
SubscriptionPlan(this.basePrice);
double get totalPrice;
}
class MonthlyPlan extends SubscriptionPlan {
MonthlyPlan(super.basePrice);
@override
double get totalPrice => basePrice;
}
class AnnualDiscountPlan extends SubscriptionPlan {
AnnualDiscountPlan(super.basePrice);
@override
double get totalPrice => basePrice * 12 * 0.8;
}
class FamilyPlan extends SubscriptionPlan {
final int members;
FamilyPlan(super.basePrice, this.members);
@override
double get totalPrice => basePrice * members;
}
void main() {
print(MonthlyPlan(300).totalPrice);
print(AnnualDiscountPlan(300).totalPrice);
print(FamilyPlan(300, 4).totalPrice);
}
```
### 16. Слои редактора
```dart
abstract class CanvasLayer {
double opacity;
bool visible;
int zIndex;
CanvasLayer({this.opacity = 1.0, this.visible = true, this.zIndex = 0});
void draw();
}
class VectorLayer extends CanvasLayer {
VectorLayer({super.opacity, super.visible, super.zIndex});
@override
void draw() => print("Векторный слой z=$zIndex");
}
class RasterBitmapLayer extends CanvasLayer {
RasterBitmapLayer({super.opacity, super.visible, super.zIndex});
@override
void draw() => print("Растровый слой z=$zIndex");
}
class TextLayer extends CanvasLayer {
final String text;
TextLayer(this.text, {super.opacity, super.visible, super.zIndex});
@override
void draw() => print("Текст '$text' z=$zIndex");
}
void main() {
final layers = <CanvasLayer>[TextLayer("Hi", zIndex: 2), VectorLayer(zIndex: 1), RasterBitmapLayer()];
layers.sort((a, b) => a.zIndex.compareTo(b.zIndex));
for (final l in layers.where((l) => l.visible)) {
l.draw();
}
}
```
### 17. Платёжные шлюзы
```dart
abstract class PaymentGateway {
bool pay(double amount);
void log(String message) => print("[${runtimeType}] $message"); // "защищённый" лог (в Dart — просто метод)
}
class SbpGateway extends PaymentGateway {
@override
bool pay(double amount) {
log("СБП платёж на $amount ₽");
return true;
}
}
class BankCardGateway extends PaymentGateway {
@override
bool pay(double amount) {
log("Оплата картой $amount ₽");
return amount < 100000;
}
}
void main() {
SbpGateway().pay(500);
BankCardGateway().pay(1500);
}
```
### 18. Онбординг
```dart
abstract class OnboardingScreen {
final String illustration;
OnboardingScreen(this.illustration);
void skip() => print("Тур пропущен");
String get text;
void render() => print("[$illustration] $text (кнопка «Пропустить»)");
}
class WelcomeStep extends OnboardingScreen {
WelcomeStep() : super("welcome.png");
@override
String get text => "Добро пожаловать!";
}
class FeaturesStep extends OnboardingScreen {
FeaturesStep() : super("features.png");
@override
String get text => "Основные возможности";
}
void main() {
WelcomeStep().render();
FeaturesStep().render();
}
```
### 19. Кнопка с состояниями
```dart
class WidgetButton {
final String label;
final void Function()? onPressed;
WidgetButton(this.label, this.onPressed);
void tap() => onPressed?.call();
}
class AsyncLoadingButton extends WidgetButton {
final Future<void> Function() asyncAction;
bool isLoading = false;
AsyncLoadingButton(String label, this.asyncAction) : super(label, null);
@override
void tap() async {
if (isLoading) return;
isLoading = true;
print("Индикатор загрузки: показан");
await asyncAction();
isLoading = false;
print("Индикатор загрузки: скрыт");
}
}
void main() {
AsyncLoadingButton("Отправить", () => Future.delayed(const Duration(seconds: 1))).tap();
}
```
### 20. Логгеры
```dart
import 'dart:io';
abstract class AppLogger {
void log(String message);
}
class ConsoleLogger extends AppLogger {
@override
void log(String message) => print("[${DateTime.now().toIso8601String()}] $message");
}
class FileCrashLogger extends AppLogger {
final String path;
FileCrashLogger(this.path);
@override
void log(String message) => File(path).writeAsStringSync("${DateTime.now()} $message\n", mode: FileMode.append);
}
void main() {
final List<AppLogger> loggers = [ConsoleLogger(), FileCrashLogger("crash.log")];
for (final l in loggers) {
l.log("Приложение запущено");
}
}
```
### Блок 3: Абстрактные контракты, интерфейсы и репозитории
### 21. ProductRepository
```dart
class Product {
final int id;
final String name;
Product(this.id, this.name);
}
abstract class ProductRepository {
Future<List<Product>> getAll();
Future<Product> getById(int id);
}
class NetworkProductRepository implements ProductRepository {
@override
Future<List<Product>> getAll() async {
await Future.delayed(const Duration(milliseconds: 300)); // имитация сети
return [Product(1, "Телефон"), Product(2, "Ноутбук")];
}
@override
Future<Product> getById(int id) async => (await getAll()).firstWhere((p) => p.id == id);
}
class MockProductRepository implements ProductRepository {
final _items = [Product(1, "Mock-товар")];
@override
Future<List<Product>> getAll() async => _items;
@override
Future<Product> getById(int id) async => _items.firstWhere((p) => p.id == id);
}
void main() async {
ProductRepository repo = MockProductRepository();
print((await repo.getById(1)).name);
}
```
### 22. BiometricAuthService
```dart
abstract class BiometricAuthService {
Future<bool> canAuthenticate();
Future<bool> authenticate();
}
class FakeBiometricAuthService implements BiometricAuthService {
final bool succeeds;
FakeBiometricAuthService({this.succeeds = true});
@override
Future<bool> canAuthenticate() async => true;
@override
Future<bool> authenticate() async => succeeds;
}
void main() async {
final s = FakeBiometricAuthService();
if (await s.canAuthenticate()) print(await s.authenticate());
}
```
### 23. CryptoService
```dart
import 'dart:convert';
abstract class CryptoService {
String encrypt(String plainText);
String decrypt(String cipherText);
}
// Учебная реализация (XOR + base64). Для настоящего AES используйте пакет `encrypt`.
class AesCryptoService implements CryptoService {
final List<int> _key;
AesCryptoService(String key) : _key = utf8.encode(key);
List<int> _xor(List<int> data) => [for (var i = 0; i < data.length; i++) data[i] ^ _key[i % _key.length]];
@override
String encrypt(String plainText) => base64.encode(_xor(utf8.encode(plainText)));
@override
String decrypt(String cipherText) => utf8.decode(_xor(base64.decode(cipherText)));
}
void main() {
final c = AesCryptoService("secret");
final e = c.encrypt("Привет");
print(e);
print(c.decrypt(e));
}
```
### 24. AnalyticsProvider
```dart
abstract class AnalyticsProvider {
void logEvent(String name, Map<String, Object> params);
}
class ConsoleAnalytics implements AnalyticsProvider {
@override
void logEvent(String name, Map<String, Object> params) => print("Console: $name $params");
}
class FirebaseLikeAnalytics implements AnalyticsProvider {
@override
void logEvent(String name, Map<String, Object> params) => print("Firebase: $name $params");
}
class CompositeAnalyticsProvider implements AnalyticsProvider {
final List<AnalyticsProvider> providers;
CompositeAnalyticsProvider(this.providers);
@override
void logEvent(String name, Map<String, Object> params) {
for (final p in providers) {
p.logEvent(name, params);
}
}
}
void main() {
CompositeAnalyticsProvider([ConsoleAnalytics(), FirebaseLikeAnalytics()]).logEvent("purchase", {"sum": 990});
}
```
### 25. Геолокатор
```dart
import 'dart:async';
class GeoPoint {
final double lat, lon;
GeoPoint(this.lat, this.lon);
@override
String toString() => "($lat, $lon)";
}
abstract class LocationService {
Future<GeoPoint> getCurrent();
Stream<GeoPoint> get positionStream;
}
class FakeLocationService implements LocationService {
@override
Future<GeoPoint> getCurrent() async => GeoPoint(55.75, 37.61);
@override
Stream<GeoPoint> get positionStream async* {
for (var i = 0; i < 5; i++) {
await Future.delayed(const Duration(milliseconds: 500));
yield GeoPoint(55.75 + i * 0.001, 37.61 + i * 0.001);
}
}
}
void main() async {
await for (final p in FakeLocationService().positionStream) {
print(p);
}
}
```
### 26. NotificationSender
```dart
abstract class NotificationSender {
void sendNow(String title, String body);
void schedule(String title, String body, DateTime when);
}
class ConsoleNotificationSender implements NotificationSender {
@override
void sendNow(String title, String body) => print("PUSH: $title — $body");
@override
void schedule(String title, String body, DateTime when) {
final delay = when.difference(DateTime.now());
Future.delayed(delay.isNegative ? Duration.zero : delay, () => sendNow(title, body));
print("Запланировано на $when");
}
}
void main() {
final s = ConsoleNotificationSender();
s.sendNow("Привет", "Мгновенное");
s.schedule("Позже", "Отложенное", DateTime.now().add(const Duration(seconds: 2)));
}
```
### 27. DataSyncEngine
```dart
abstract class DataSyncEngine {
Future<List<String>> fetchRemoteDelta();
Future<void> applyLocalChanges(List<String> changes);
Future<void> sync() async {
final delta = await fetchRemoteDelta();
await applyLocalChanges(delta);
}
}
class SimpleSyncEngine extends DataSyncEngine {
final List<String> local = [];
@override
Future<List<String>> fetchRemoteDelta() async => ["запись 1", "запись 2"];
@override
Future<void> applyLocalChanges(List<String> changes) async => local.addAll(changes);
}
void main() async {
final e = SimpleSyncEngine();
await e.sync();
print(e.local);
}
```
### 28. ConnectivityService
```dart
import 'dart:async';
abstract class ConnectivityService {
bool get isOnline;
Stream<bool> get onStatusChanged;
Stream<void> get onConnectionLost => onStatusChanged.where((online) => !online);
}
class FakeConnectivityService extends ConnectivityService {
final _controller = StreamController<bool>.broadcast();
bool _online = true;
@override
bool get isOnline => _online;
@override
Stream<bool> get onStatusChanged => _controller.stream;
void setOnline(bool value) {
_online = value;
_controller.add(value);
}
}
void main() {
final c = FakeConnectivityService();
c.onConnectionLost.listen((_) => print("Интернет потерян!"));
c.setOnline(false);
}
```
### 29. KeyValueStorage
```dart
abstract class KeyValueStorage {
T? read<T>(String key);
Future<void> write<T>(String key, T value);
}
class InMemoryKeyValueStorage implements KeyValueStorage {
final Map<String, Object?> _data = {};
@override
T? read<T>(String key) {
final v = _data[key];
return v is T ? v : null;
}
@override
Future<void> write<T>(String key, T value) async => _data[key] = value;
}
void main() async {
final s = InMemoryKeyValueStorage();
await s.write<int>("age", 20);
print(s.read<int>("age")); // 20
print(s.read<String>("age")); // null
}
```
### 30. ReportExporter
```dart
abstract class ReportExporter {
String get extension;
List<int> export(List<List<String>> rows);
}
class PdfReportExporter implements ReportExporter {
@override
String get extension => "pdf";
@override
List<int> export(List<List<String>> rows) => "%PDF-FAKE\n${rows.map((r) => r.join(' | ')).join('\n')}".codeUnits;
}
class ExcelReportExporter implements ReportExporter {
@override
String get extension => "csv";
@override
List<int> export(List<List<String>> rows) => rows.map((r) => r.join(";")).join("\n").codeUnits;
}
void main() {
final rows = [["Имя", "Сумма"], ["Аня", "100"]];
for (final e in [PdfReportExporter(), ExcelReportExporter()]) {
print("report.${e.extension}: ${e.export(rows).length} байт");
}
}
```
### Блок 4: Миксины
### 31. DebounceMixin
```dart
mixin DebounceMixin {
final Map<String, DateTime> _last = {};
bool debounce(String actionId, void Function() action, {Duration delay = const Duration(milliseconds: 500)}) {
final now = DateTime.now();
final last = _last[actionId];
if (last != null && now.difference(last) < delay) return false;
_last[actionId] = now;
action();
return true;
}
}
class LikeButton with DebounceMixin {
void tap() => debounce("like", () => print("Лайк!"));
}
void main() async {
final b = LikeButton();
b.tap();
b.tap(); // проигнорирован
await Future.delayed(const Duration(milliseconds: 600));
b.tap();
}
```
### 32. Восстановление текста полей (RestorationMixin)
```dart
import 'package:flutter/widgets.dart';
// Простое решение: при сворачивании приложения тексты сохраняются, при возврате — восстанавливаются.
mixin TextRestorationMixin<T extends StatefulWidget> on State<T>, WidgetsBindingObserver {
static final Map<String, String> _saved = {};
final Map<String, TextEditingController> _controllers = {};
TextEditingController restorableController(String id) =>
_controllers.putIfAbsent(id, () => TextEditingController(text: _saved[id] ?? ""));
void saveTexts() => _controllers.forEach((id, c) => _saved[id] = c.text);
@override
void initState() {
super.initState();
WidgetsBinding.instance.addObserver(this);
}
@override
void didChangeAppLifecycleState(AppLifecycleState state) {
if (state == AppLifecycleState.paused || state == AppLifecycleState.inactive) saveTexts();
}
@override
void dispose() {
saveTexts();
WidgetsBinding.instance.removeObserver(this);
for (final c in _controllers.values) {
c.dispose();
}
super.dispose();
}
}
// Использование: class _MyState extends State<MyPage> with WidgetsBindingObserver, TextRestorationMixin
```
### 33. NetworkAwareMixin
```dart
abstract class NetworkChecker {
Future<bool> get isOnline;
}
class FakeNetworkChecker implements NetworkChecker {
@override
Future<bool> get isOnline async => true;
}
mixin NetworkAwareMixin {
NetworkChecker get networkChecker;
Future<T> withNetwork<T>(Future<T> Function() request) async {
if (!await networkChecker.isOnline) {
throw Exception("Нет подключения к интернету");
}
return request();
}
}
class ProfilePresenter with NetworkAwareMixin {
@override
final NetworkChecker networkChecker = FakeNetworkChecker();
Future<String> loadProfile() => withNetwork(() async => "Профиль загружен");
}
void main() async => print(await ProfilePresenter().loadProfile());
```
### 34. CurrencyFormatterMixin
```dart
mixin CurrencyFormatterMixin {
String formatPrice(num value, {String symbol = "₽"}) {
final parts = value.toStringAsFixed(2).split(".");
final intPart = parts[0].replaceAllMapped(RegExp(r"\B(?=(\d{3})+(?!\d))"), (m) => " ");
return "$intPart,${parts[1]} $symbol";
}
}
class CartScreen with CurrencyFormatterMixin {
void show() => print(formatPrice(1234567.5));
}
void main() => CartScreen().show(); // 1 234 567,50 ₽
```
### 35. PeriodicRefreshMixin
```dart
import 'dart:async';
mixin PeriodicRefreshMixin {
Timer? _timer;
Future<void> refresh();
void startAutoRefresh({Duration every = const Duration(seconds: 30)}) {
_timer?.cancel();
_timer = Timer.periodic(every, (_) => refresh());
}
void stopAutoRefresh() {
_timer?.cancel();
_timer = null;
}
}
class NewsScreen with PeriodicRefreshMixin {
@override
Future<void> refresh() async => print("Опрос сервера: ${DateTime.now()}");
}
void main() async {
final s = NewsScreen()..startAutoRefresh(every: const Duration(seconds: 1));
await Future.delayed(const Duration(seconds: 3));
s.stopAutoRefresh();
}
```
### 36. PassportValidationMixin
```dart
mixin PassportValidationMixin {
bool validateSeries(String s) => RegExp(r"^\d{4}$").hasMatch(s);
bool validateNumber(String n) => RegExp(r"^\d{6}$").hasMatch(n);
bool validateDepartmentCode(String c) => RegExp(r"^\d{3}-\d{3}$").hasMatch(c);
bool validatePassport(String series, String number, String code) =>
validateSeries(series) && validateNumber(number) && validateDepartmentCode(code);
}
class KycForm with PassportValidationMixin {}
void main() {
final f = KycForm();
print(f.validatePassport("4510", "123456", "770-001")); // true
print(f.validatePassport("45", "123456", "770001")); // false
}
```
### 37. Fade-анимация
```dart
import 'package:flutter/material.dart';
mixin FadeMixin<T extends StatefulWidget> on State<T>, SingleTickerProviderStateMixin<T> {
late final AnimationController fadeController;
double get opacity => fadeController.value; // от 0.0 до 1.0
void initFade({Duration duration = const Duration(milliseconds: 500)}) {
fadeController = AnimationController(vsync: this, duration: duration)
..addListener(() => setState(() {}));
}
void fadeIn() => fadeController.forward(from: 0.0);
void fadeOut() => fadeController.reverse(from: 1.0);
void disposeFade() => fadeController.dispose();
}
class _DemoState extends State<Demo> with SingleTickerProviderStateMixin, FadeMixin {
@override
void initState() {
super.initState();
initFade();
fadeIn();
}
@override
void dispose() {
disposeFade();
super.dispose();
}
@override
Widget build(BuildContext context) => Opacity(opacity: opacity, child: const Text("Привет"));
}
class Demo extends StatefulWidget {
const Demo({super.key});
@override
State<Demo> createState() => _DemoState();
}
```
### 38. Переключение темы
```dart
import 'package:flutter/material.dart';
mixin ThemeAccessMixin<T extends StatefulWidget> on State<T> {
ThemeData get theme => Theme.of(context);
ColorScheme get colors => theme.colorScheme;
bool get isDark => theme.brightness == Brightness.dark;
Color get textColor => isDark ? Colors.white : Colors.black87;
}
```
### 39. DeepLinkHandlerMixin
```dart
mixin DeepLinkHandlerMixin {
void openScreen(String route, Map<String, String> params);
bool handleDeepLink(String link) {
final uri = Uri.tryParse(link);
if (uri == null || uri.scheme != "myapp") return false;
// myapp://product?id=42 -> host = "product"
openScreen("/${uri.host}", uri.queryParameters);
return true;
}
}
class Router with DeepLinkHandlerMixin {
@override
void openScreen(String route, Map<String, String> params) => print("Открыт $route $params");
}
void main() => Router().handleDeepLink("myapp://product?id=42");
```
### 40. Аудит действий администратора
```dart
class SecurityServer {
void send(Map<String, Object> record) => print("-> сервер безопасности: $record");
}
mixin AdminAuditMixin {
SecurityServer get securityServer;
String get adminId;
void audit(String action, {Map<String, Object> details = const {}}) {
securityServer.send({
"admin": adminId,
"action": action,
"time": DateTime.now().toIso8601String(),
...details,
});
}
}
class Moderator with AdminAuditMixin {
@override
final SecurityServer securityServer = SecurityServer();
@override
final String adminId = "mod-7";
void banUser(String userId) {
audit("ban_user", details: {"target": userId});
print("Пользователь $userId заблокирован");
}
}
void main() => Moderator().banUser("user-123");
```
### Блок 5: Модификаторы классов и архитектурные состояния
### 41. CartState
```dart
class Item {
final String name;
final double price;
Item(this.name, this.price);
}
sealed class CartState {}
class CartEmpty extends CartState {}
class CartLoading extends CartState {}
class CartLoaded extends CartState {
final List<Item> items;
final double total;
CartLoaded(this.items, this.total);
}
class CartCheckoutError extends CartState {
final String error;
CartCheckoutError(this.error);
}
String handleCart(CartState s) => switch (s) {
CartEmpty() => "Корзина пуста",
CartLoading() => "Загрузка...",
CartLoaded(:final items, :final total) => "Товаров: ${items.length}, итого: $total ₽",
CartCheckoutError(:final error) => "Ошибка оформления: $error",
};
void main() {
print(handleCart(CartEmpty()));
print(handleCart(CartLoaded([Item("Чай", 150)], 150)));
}
```
### 42. Result<T>
```dart
sealed class Result<T> {}
class Success<T> extends Result<T> {
final T data;
Success(this.data);
}
class Failure<T> extends Result<T> {
final Exception exception;
Failure(this.exception);
}
String show<T>(Result<T> r) => switch (r) {
Success(:final data) => "Данные: $data",
Failure(:final exception) => "Ошибка: $exception",
};
void main() {
print(show(Success<int>(42)));
print(show(Failure<int>(Exception("Сеть недоступна"))));
}
```
### 43. LoginEvent
```dart
sealed class LoginEvent {}
class LoginEmailChanged extends LoginEvent {
final String email;
LoginEmailChanged(this.email);
}
class LoginPasswordChanged extends LoginEvent {
final String pass;
LoginPasswordChanged(this.pass);
}
class LoginSubmitted extends LoginEvent {}
class LoginBloc {
String email = "", pass = "";
void add(LoginEvent e) {
switch (e) {
case LoginEmailChanged(:final email): this.email = email;
case LoginPasswordChanged(:final pass): this.pass = pass;
case LoginSubmitted(): print("Отправка: $email / ${'*' * pass.length}");
}
}
}
void main() {
final b = LoginBloc();
b.add(LoginEmailChanged("a@b.ru"));
b.add(LoginPasswordChanged("12345678"));
b.add(LoginSubmitted());
}
```
### 44. final class EncryptionKey
```dart
// key.dart
final class EncryptionKey {
final List<int> bytes;
const EncryptionKey(this.bytes);
}
// В другой библиотеке — ошибки компиляции:
// class Fake extends EncryptionKey { Fake() : super([]); }
// class Fake2 implements EncryptionKey { ... }
```
### 45. interface class для SDK
```dart
// sdk.dart
interface class PaymentTerminal {
bool processPayment(double amount) {
print("Платёж $amount обработан");
return true;
}
}
// У клиента SDK:
// class MyTerminal implements PaymentTerminal {
// @override bool processPayment(double amount) => false;
// }
// class Bad extends PaymentTerminal {} // ОШИБКА: extends запрещён вне библиотеки
```
### 46. AppRoute
```dart
sealed class AppRoute {}
class HomeRoute extends AppRoute {}
class ProductRoute extends AppRoute {
final int productId;
ProductRoute(this.productId);
}
class ProfileRoute extends AppRoute {
final String userId;
ProfileRoute(this.userId);
}
class SettingsRoute extends AppRoute {}
String pathOf(AppRoute r) => switch (r) {
HomeRoute() => "/",
ProductRoute(:final productId) => "/product/$productId",
ProfileRoute(:final userId) => "/profile/$userId",
SettingsRoute() => "/settings",
};
void main() {
print(pathOf(ProductRoute(5)));
print(pathOf(SettingsRoute()));
}
```
### 47. Паттерн «Команда»
```dart
abstract class Command {
void execute();
void undo();
}
class TextEditor {
String text = "";
}
class AppendTextCommand implements Command {
final TextEditor editor;
final String added;
AppendTextCommand(this.editor, this.added);
@override
void execute() => editor.text += added;
@override
void undo() => editor.text = editor.text.substring(0, editor.text.length - added.length);
}
class CommandHistory {
final List<Command> _done = [];
void run(Command c) {
c.execute();
_done.add(c);
}
void undo() {
if (_done.isNotEmpty) _done.removeLast().undo();
}
}
void main() {
final e = TextEditor();
final h = CommandHistory();
h.run(AppendTextCommand(e, "Привет"));
h.run(AppendTextCommand(e, ", мир"));
print(e.text); // Привет, мир
h.undo();
print(e.text); // Привет
}
```
### 48. NotificationBuilder
```dart
class AppNotification {
final String title, body, icon;
AppNotification._(this.title, this.body, this.icon);
@override
String toString() => "[$icon] $title: $body";
}
class NotificationBuilder {
String _title = "", _body = "", _icon = "info";
NotificationBuilder setTitle(String t) {
_title = t;
return this;
}
NotificationBuilder setBody(String b) {
_body = b;
return this;
}
NotificationBuilder setIcon(String i) {
_icon = i;
return this;
}
AppNotification build() {
if (_title.isEmpty) throw StateError("Заголовок обязателен");
return AppNotification._(_title, _body, _icon);
}
}
void main() {
print(NotificationBuilder().setTitle("Заказ").setBody("Готов к выдаче").setIcon("bag").build());
}
```
### 49. Observer / EventBus
```dart
typedef Listener<T> = void Function(T event);
class EventBus {
final Map<Type, List<Function>> _listeners = {};
void subscribe<T>(Listener<T> listener) => _listeners.putIfAbsent(T, () => []).add(listener);
void unsubscribe<T>(Listener<T> listener) => _listeners[T]?.remove(listener);
void publish<T>(T event) {
for (final l in List<Function>.from(_listeners[T] ?? [])) {
(l as Listener<T>)(event);
}
}
}
class UserLoggedIn {
final String name;
UserLoggedIn(this.name);
}
class OrderCreated {
final int id;
OrderCreated(this.id);
}
void main() {
final bus = EventBus();
void onLogin(UserLoggedIn e) => print("Вошёл ${e.name}");
bus.subscribe<UserLoggedIn>(onLogin);
bus.subscribe<OrderCreated>((e) => print("Заказ №${e.id}"));
bus.publish(UserLoggedIn("Аня"));
bus.publish(OrderCreated(7));
bus.unsubscribe<UserLoggedIn>(onLogin);
bus.publish(UserLoggedIn("Борис")); // никто не слушает
}
```
### 50. Модуль заказа такси
```dart
import 'dart:math';
sealed class TaxiOrderState {}
class SearchingCar extends TaxiOrderState {}
class CarAssigned extends TaxiOrderState {
final String driver, plate;
CarAssigned(this.driver, this.plate);
}
class InTrip extends TaxiOrderState {
final double distanceKm;
InTrip(this.distanceKm);
}
class Completed extends TaxiOrderState {
final double fare;
Completed(this.fare);
}
class Cancelled extends TaxiOrderState {
final String reason;
Cancelled(this.reason);
}
mixin LocationTrackingMixin {
final List<(double, double)> track = [];
void updateLocation(double lat, double lon) => track.add((lat, lon));
double get distanceKm {
double sum = 0;
for (var i = 1; i < track.length; i++) {
final dLat = (track[i].$1 - track[i - 1].$1) * 111;
final dLon = (track[i].$2 - track[i - 1].$2) * 111 * cos(track[i].$1 * pi / 180);
sum += sqrt(dLat * dLat + dLon * dLon);
}
return sum;
}
}
mixin FareCalculationMixin {
double fare(double km, {double base = 100, double perKm = 25}) => base + km * perKm;
}
class TaxiOrder with LocationTrackingMixin, FareCalculationMixin {
final String from, to;
TaxiOrderState state;
TaxiOrder._(this.from, this.to, this.state);
TaxiOrder.create(String from, String to) : this._(from, to, SearchingCar());
factory TaxiOrder.preassigned(String from, String to, String driver, String plate) =>
TaxiOrder._(from, to, CarAssigned(driver, plate));
}
class TripController {
final TaxiOrder order;
TripController(this.order);
void next() {
order.state = switch (order.state) {
SearchingCar() => CarAssigned("Иван", "А123ВС"),
CarAssigned() => InTrip(0),
InTrip() => Completed(order.fare(order.distanceKm)),
Completed() => order.state,
Cancelled() => order.state,
};
print(describe());
}
void cancel(String reason) {
if (order.state is! Completed) order.state = Cancelled(reason);
print(describe());
}
String describe() => switch (order.state) {
SearchingCar() => "Поиск машины...",
CarAssigned(:final driver, :final plate) => "Едет $driver ($plate)",
InTrip(:final distanceKm) => "В пути, ${distanceKm.toStringAsFixed(1)} км",
Completed(:final fare) => "Поездка завершена. К оплате: ${fare.toStringAsFixed(0)} ₽",
Cancelled(:final reason) => "Отменено: $reason",
};
}
void main() {
final order = TaxiOrder.create("Дом", "Работа");
final c = TripController(order);
c.next();
c.next();
order.updateLocation(55.75, 37.61);
order.updateLocation(55.79, 37.65);
c.next();
}
```
