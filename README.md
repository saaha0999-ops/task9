# Практическая работа №9: Асинхронное программирование в Dart — решения
> Каждое решение — отдельный фрагмент. Имена функций и классов повторяются, поэтому запускай каждое в своём `.dart`-файле (или DartPad; изоляты и `dart:io` в DartPad не работают — для них нужен `dart run`).
---
## Тема 1. Future и async/await
### 1. Симулятор авторизации
```dart
class AuthException implements Exception {
final String message;
AuthException(this.message);
@override
String toString() => "AuthException: $message";
}
Future<bool> login(String user, String pass) async {
await Future.delayed(const Duration(milliseconds: 1500));
if (pass == "1234") return true;
throw AuthException("Неверный пароль для пользователя $user");
}
void main() async {
print(await login("anna", "1234"));
try {
await login("anna", "0000");
} on AuthException catch (e) {
print(e);
}
}
```
### 2. Параллельное чтение конфигов
```dart
Future<MapEntry<String, String>> readConfig(String name, int ms) async {
await Future.delayed(Duration(milliseconds: ms));
return MapEntry(name, "содержимое $name");
}
void main() async {
final sw = Stopwatch()..start();
final entries = await Future.wait([
readConfig("app.json", 800),
readConfig("theme.json", 500),
readConfig("user.json", 1200),
]);
final config = Map.fromEntries(entries);
print(config);
print("Время: ${sw.elapsedMilliseconds} мс"); // ~1200, а не 2500
}
```
### 3. Future.any — гонка зеркал
```dart
Future<String> mirror(String name, int ms) async {
await Future.delayed(Duration(milliseconds: ms));
return "Ответ от $name";
}
void main() async {
final fastest = await Future.any([mirror("mirror-1", 800), mirror("mirror-2", 1200)]);
print(fastest); // mirror-1
}
```
### 4. Retry
```dart
Future<T> retry<T>(Future<T> Function() task, int maxAttempts) async {
for (var attempt = 1; ; attempt++) {
try {
return await task();
} catch (e) {
print("Попытка $attempt провалилась: $e");
if (attempt >= maxAttempts) rethrow;
await Future.delayed(const Duration(seconds: 1));
}
}
}
int _calls = 0;
Future<String> flaky() async {
_calls++;
if (_calls < 3) throw Exception("Сбой сети");
return "Успех с попытки $_calls";
}
void main() async => print(await retry(flaky, 3));
```
### 5. Цепочка then / catchError / whenComplete
```dart
class AuthException implements Exception {
final String message;
AuthException(this.message);
@override
String toString() => "AuthException: $message";
}
Future<bool> login(String user, String pass) => Future.delayed(const Duration(milliseconds: 1500), () {
if (pass == "1234") return true;
throw AuthException("Неверный пароль");
});
void main() {
login("anna", "0000")
.then((ok) => print("Вход выполнен: $ok"))
.catchError((e) => print("Ошибка: $e"))
.whenComplete(() => print("Скрываем индикатор загрузки"));
}
```
---
## Тема 2. Stream и async*
### 1. Секундомер
```dart
Stream<int> stopWatch() async* {
var ticks = 0;
while (true) {
await Future.delayed(const Duration(seconds: 1));
yield ++ticks;
}
}
void main() async {
await for (final t in stopWatch().take(5)) {
print("Прошло $t с");
}
}
```
### 2. Поток GPS-координат
```dart
import 'dart:math';
Stream<(double, double)> gpsStream() async* {
final rnd = Random();
var lat = 55.751244, lon = 37.618423;
while (true) {
await Future.delayed(const Duration(milliseconds: 500));
lat += (rnd.nextDouble() - 0.5) * 0.0004;
lon += (rnd.nextDouble() - 0.5) * 0.0004;
yield (lat, lon);
}
}
void main() async {
await for (final (lat, lon) in gpsStream().take(6)) {
print("lat=${lat.toStringAsFixed(6)}, lon=${lon.toStringAsFixed(6)}");
}
}
```
### 3. Broadcast-стрим котировок
```dart
Stream<double> prices() async* {
var p = 100.0;
for (var i = 0; i < 5; i++) {
await Future.delayed(const Duration(milliseconds: 300));
p += 1.5;
yield p;
}
}
void main() async {
final stream = prices().asBroadcastStream();
stream.listen((p) => print("UI: цена $p"));
stream.listen((p) => print("Аналитика: записали $p"));
await Future.delayed(const Duration(seconds: 3));
}
```
### 4. fold
```dart
void main() async {
final numbers = Stream.fromIterable(List.generate(10, (i) => i + 1));
final sum = await numbers.fold<int>(0, (acc, x) => acc + x);
print("Сумма: $sum"); // 55
}
```
### 5. yield*
```dart
Stream<int> first() async* {
for (var i = 1; i <= 3; i++) {
await Future.delayed(const Duration(milliseconds: 200));
yield i;
}
}
Stream<int> second() async* {
for (var i = 10; i <= 12; i++) {
await Future.delayed(const Duration(milliseconds: 200));
yield i;
}
}
Stream<int> combined() async* {
yield* first();
yield* second();
}
void main() async {
await for (final v in combined()) {
print(v); // 1 2 3 10 11 12
}
}
```
---
## Тема 3. StreamController и StreamTransformer
### 1. Трансформер в верхний регистр
```dart
import 'dart:async';
void main() async {
final upper = StreamTransformer<String, String>.fromHandlers(
handleData: (data, sink) => sink.add(data.toUpperCase()),
);
await Stream.fromIterable(["привет", "hello", "dart"]).transform(upper).forEach(print);
}
```
### 2. SearchControllerWrapper
```dart
import 'dart:async';
class SearchControllerWrapper {
final _input = StreamController<String>();
final _output = StreamController<String>.broadcast();
SearchControllerWrapper() {
_input.stream.map((s) => s.trim().toLowerCase()).where((s) => s.isNotEmpty).listen(_output.add);
}
Sink<String> get sink => _input.sink;
Stream<String> get stream => _output.stream;
void dispose() {
_input.close();
_output.close();
}
}
void main() async {
final c = SearchControllerWrapper();
c.stream.listen((q) => print("Запрос: '$q'"));
c.sink.add("  Flutter ");
c.sink.add("   ");
c.sink.add("DART");
await Future.delayed(const Duration(milliseconds: 100));
c.dispose();
}
```
### 3. Ошибки в контроллере
```dart
import 'dart:async';
void main() async {
final c = StreamController<int>();
c.stream.listen(
(v) => print("Данные: $v"),
onError: (e) => print("Перехвачена ошибка: $e"),
);
c.sink.add(1);
c.sink.addError(Exception("Сбой датчика"));
c.sink.add(2);
await Future.delayed(const Duration(milliseconds: 100));
await c.close();
}
```
### 4. isClosed
```dart
import 'dart:async';
void safeAdd<T>(StreamController<T> c, T value) {
if (c.isClosed) {
print("Контроллер закрыт, событие $value пропущено");
return;
}
c.add(value);
}
void main() async {
final c = StreamController<int>();
c.stream.listen(print);
safeAdd(c, 1);
await c.close();
safeAdd(c, 2); // без исключения
}
```
### 5. onListen / onCancel
```dart
import 'dart:async';
void main() async {
var subscribers = 0;
late StreamController<int> c;
c = StreamController<int>.broadcast(
onListen: () => print("Появился подписчик (всего: ${++subscribers})"),
onCancel: () => print("Подписчик ушёл (осталось: ${--subscribers})"),
);
// Для broadcast onListen/onCancel вызываются при первом/последнем подписчике
final s1 = c.stream.listen((_) {});
await s1.cancel();
final s2 = c.stream.listen((_) {});
await s2.cancel();
await c.close();
}
```
---
## Тема 4. Изоляты
### 1. Парсер JSON в Isolate.run
```dart
import 'dart:convert';
import 'dart:isolate';
class Record {
final int id;
final String name;
Record(this.id, this.name);
}
List<Record> parseRecords(String json) =>
(jsonDecode(json) as List).map((e) => Record(e["id"] as int, e["name"] as String)).toList();
void main() async {
final json = jsonEncode(List.generate(10000, (i) => {"id": i, "name": "Запись $i"}));
final records = await Isolate.run(() => parseRecords(json));
print("Разобрано: ${records.length}, последняя: ${records.last.name}");
}
```
### 2. Простые числа в фоне
```dart
import 'dart:async';
import 'dart:io';
import 'dart:isolate';
int countPrimes(int limit) {
final sieve = List<bool>.filled(limit + 1, true);
sieve[0] = sieve[1] = false;
for (var i = 2; i * i <= limit; i++) {
if (sieve[i]) {
for (var j = i * i; j <= limit; j += i) {
sieve[j] = false;
}
}
}
return sieve.where((p) => p).length;
}
void main() async {
final timer = Timer.periodic(const Duration(seconds: 1), (_) => stdout.write("."));
final count = await Isolate.run(() => countPrimes(1000000));
timer.cancel();
print("\nПростых чисел до 1 000 000: $count");
}
```
### 3. Двусторонний обмен через порты
```dart
import 'dart:isolate';
void calculatorWorker(SendPort mainPort) {
final workerPort = ReceivePort();
mainPort.send(workerPort.sendPort);
workerPort.listen((message) {
if (message == "exit") {
workerPort.close();
return;
}
final [a, b, replyTo] = message as List;
(replyTo as SendPort).send((a as num) + (b as num));
});
}
void main() async {
final mainPort = ReceivePort();
await Isolate.spawn(calculatorWorker, mainPort.sendPort);
final SendPort workerSend = await mainPort.first; // воркер сразу присылает свой SendPort
final replies = ReceivePort();
workerSend.send([2, 3, replies.sendPort]);
print("2 + 3 = ${await replies.first}");
workerSend.send("exit");
}
```
### 4. Замер «просадки» отзывчивости
```dart
import 'dart:async';
import 'dart:isolate';
int heavyLoop(int n) {
var s = 0;
for (var i = 0; i < n; i++) {
s += i & 1;
}
return s;
}
Future<int> measure(Future<void> Function() job) async {
var ticks = 0;
final timer = Timer.periodic(const Duration(milliseconds: 100), (_) => ticks++);
await job();
timer.cancel();
return ticks;
}
void main() async {
const n = 2000000000;
final syncTicks = await measure(() async {
await Future.delayed(Duration.zero);
heavyLoop(n); // блокирует Event Loop
});
final isolateTicks = await measure(() => Isolate.run(() => heavyLoop(n)));
print("Тиков таймера при синхронном цикле: $syncTicks");
print("Тиков таймера при Isolate.run: $isolateTicks");
}
```
### 5. isolate.kill
```dart
import 'dart:isolate';
void infiniteWorker(SendPort port) {
var i = 0;
while (true) {
i++;
if (i % 500000000 == 0) port.send("работаю... $i");
}
}
void main() async {
final receive = ReceivePort();
final isolate = await Isolate.spawn(infiniteWorker, receive.sendPort);
receive.listen((m) => print(m));
await Future.delayed(const Duration(seconds: 3));
isolate.kill(priority: Isolate.immediate);
print("Изолят уничтожен");
receive.close();
}
```
---
## 50 смешанных заданий
### Блок 1: Сетевые запросы, таймеры и Future
### 1. Имитатор HTTP GET
```dart
import 'dart:convert';
class HttpException2 implements Exception {
final int status;
HttpException2(this.status);
@override
String toString() => "HTTP $status";
}
Future<Map<String, dynamic>> fetchData(String url) async {
await Future.delayed(const Duration(milliseconds: 1200));
final status = url.contains("missing") ? 404 : url.contains("error") ? 500 : 200;
switch (status) {
case 200:
return jsonDecode('{"url": "$url", "data": [1, 2, 3]}') as Map<String, dynamic>;
case 404:
throw HttpException2(404);
default:
throw HttpException2(500);
}
}
void main() async {
for (final u in ["https://api/ok", "https://api/missing", "https://api/error"]) {
try {
print(await fetchData(u));
} on HttpException2 catch (e) {
print("$u -> ошибка $e");
}
}
}
```
### 2. Exponential Backoff
```dart
import 'dart:io';
Future<T> withBackoff<T>(Future<T> Function() request, {int maxRetries = 4}) async {
var delay = const Duration(seconds: 1);
for (var attempt = 0; ; attempt++) {
try {
return await request();
} on SocketException catch (e) {
if (attempt >= maxRetries) rethrow;
print("Сбой ($e). Повтор через ${delay.inSeconds} с");
await Future.delayed(delay);
delay *= 2; // 1, 2, 4, 8
}
}
}
var _n = 0;
Future<String> request() async {
if (++_n < 4) throw const SocketException("Нет соединения");
return "Получено с попытки $_n";
}
void main() async => print(await withBackoff(request));
```
### 3. Параллельная загрузка с лимитом 2
```dart
import 'dart:async';
Future<String> fetchMeta(int i) async {
await Future.delayed(Duration(milliseconds: 500 + i * 100));
print("Метаданные картинки $i получены");
return "meta_$i";
}
Future<List<T>> waitLimited<T>(List<Future<T> Function()> tasks, int limit) async {
final results = List<T?>.filled(tasks.length, null);
var next = 0;
Future<void> worker() async {
while (next < tasks.length) {
final i = next++;
results[i] = await tasks[i]();
}
}
await Future.wait(List.generate(limit, (_) => worker()));
return results.cast<T>();
}
void main() async {
final tasks = List.generate(5, (i) => () => fetchMeta(i));
print(await waitLimited(tasks, 2));
}
```
### 4. Ping зеркал (первый успешный)
```dart
import 'dart:async';
Future<String> ping(String url, int ms, {bool fail = false}) async {
await Future.delayed(Duration(milliseconds: ms));
if (fail) throw Exception("$url недоступен");
return url;
}
Future<String> firstAlive(List<Future<String>> pings) {
final c = Completer<String>();
var failed = 0;
for (final p in pings) {
p.then((url) {
if (!c.isCompleted) c.complete(url);
}).catchError((_) {
if (++failed == pings.length && !c.isCompleted) c.completeError("Все зеркала недоступны");
});
}
return c.future;
}
void main() async {
final url = await firstAlive([
ping("mirror-a.com", 100, fail: true),
ping("mirror-b.com", 600),
ping("mirror-c.com", 300),
]);
print("Первый ответил: $url"); // mirror-c.com
}
```
### 5. Таймаут сохранения профиля
```dart
import 'dart:async';
Future<void> saveProfile() => Future.delayed(const Duration(seconds: 10), () => print("Сохранено"));
void main() async {
try {
await saveProfile().timeout(const Duration(seconds: 3));
} on TimeoutException {
print("Сохранение прервано по таймауту (3 с)");
}
}
```
### 6. Async Queue (FIFO)
```dart
import 'dart:async';
class AsyncQueue {
Future<void> _tail = Future.value();
Future<T> add<T>(Future<T> Function() task) {
final result = _tail.then((_) => task());
_tail = result.then((_) {}, onError: (_) {});
return result;
}
}
void main() async {
final q = AsyncQueue();
final futures = [
q.add(() async {
await Future.delayed(const Duration(milliseconds: 600));
print("Задача 1");
}),
q.add(() async {
await Future.delayed(const Duration(milliseconds: 100));
print("Задача 2");
}),
q.add(() async => print("Задача 3")),
];
await Future.wait(futures); // порядок 1, 2, 3
}
```
### 7. benchmark
```dart
Future<int> benchmark(Future Function() task) async {
final sw = Stopwatch()..start();
await task();
sw.stop();
print("Выполнено за ${sw.elapsedMilliseconds} мс");
return sw.elapsedMilliseconds;
}
void main() async {
await benchmark(() => Future.delayed(const Duration(milliseconds: 750)));
}
```
### 8. Ленивый асинхронный синглтон
```dart
class AppConfig {
final String apiUrl;
AppConfig._(this.apiUrl);
static Future<AppConfig>? _future;
static Future<AppConfig> getInstance() => _future ??= _load();
static Future<AppConfig> _load() async {
print("Чтение конфигурации (один раз)...");
await Future.delayed(const Duration(milliseconds: 500));
return AppConfig._("https://api.example.com");
}
}
void main() async {
final a = await AppConfig.getInstance();
final b = await AppConfig.getInstance();
print(identical(a, b)); // true
}
```
### 9. Локальные данные против сети (300 мс)
```dart
Future<String> readLocal() async {
await Future.delayed(const Duration(milliseconds: 50));
return "локальные данные (SQLite)";
}
Future<String> fetchNetwork() async {
await Future.delayed(const Duration(milliseconds: 800));
return "данные с сервера";
}
Future<String> loadData() async {
final local = readLocal();
try {
return await fetchNetwork().timeout(const Duration(milliseconds: 300));
} catch (_) {
return await local;
}
}
void main() async => print(await loadData()); // локальные данные (SQLite)
```
### 10. Debounce
```dart
import 'dart:async';
class Debouncer {
final Duration delay;
Timer? _timer;
Debouncer({this.delay = const Duration(milliseconds: 400)});
void run(void Function() action) {
_timer?.cancel();
_timer = Timer(delay, action);
}
void dispose() => _timer?.cancel();
}
void main() async {
final d = Debouncer();
for (final q in ["f", "fl", "flu", "flut"]) {
d.run(() => print("Поиск: $q"));
await Future.delayed(const Duration(milliseconds: 100));
}
await Future.delayed(const Duration(seconds: 1)); // выполнится только "flut"
}
```
### Блок 2: Stream API
### 11. Котировки и всплески более 5%
```dart
import 'dart:math';
Stream<double> btcPrice() async* {
final rnd = Random();
var price = 60000.0;
while (true) {
await Future.delayed(const Duration(milliseconds: 200));
price *= 1 + (rnd.nextDouble() - 0.5) * 0.14; // до ±7%
yield price;
}
}
Stream<String> spikes(Stream<double> prices) async* {
double? prev;
await for (final p in prices) {
if (prev != null) {
final change = (p - prev) / prev * 100;
if (change.abs() > 5) yield "Всплеск ${change.toStringAsFixed(1)}%: ${p.toStringAsFixed(0)}";
}
prev = p;
}
}
void main() async {
await for (final s in spikes(btcPrice()).take(3)) {
print(s);
}
}
```
### 12. Прогресс скачивания
```dart
Stream<int> downloadProgress() async* {
for (var p = 0; p <= 100; p += 10) {
await Future.delayed(const Duration(milliseconds: 300));
yield p;
}
}
void main() async {
await for (final p in downloadProgress()) {
print("Загрузка: $p%");
}
}
```
### 13. Merge потоков навигации
```dart
import 'dart:async';
Stream<T> merge<T>(List<Stream<T>> streams) {
late StreamController<T> c;
final subs = <StreamSubscription<T>>[];
c = StreamController<T>(
onListen: () {
var done = 0;
for (final s in streams) {
subs.add(s.listen(c.add, onError: c.addError, onDone: () {
if (++done == streams.length) c.close();
}));
}
},
onCancel: () => Future.wait(subs.map((s) => s.cancel())),
);
return c.stream;
}
Stream<String> geo() => Stream.periodic(const Duration(milliseconds: 300), (i) => "GPS #$i").take(4);
Stream<String> compass() => Stream.periodic(const Duration(milliseconds: 500), (i) => "Компас ${i * 10}°").take(3);
void main() async {
await for (final e in merge([geo(), compass()])) {
print("Навигация: $e");
}
}
```
### 14. Throttle
```dart
Stream<T> throttle<T>(Stream<T> source, Duration interval) async* {
final sw = Stopwatch();
await for (final e in source) {
if (!sw.isRunning || sw.elapsed >= interval) {
sw..reset()..start();
yield e;
}
}
}
void main() async {
final clicks = Stream.periodic(const Duration(milliseconds: 200), (i) => i).take(15);
await for (final c in throttle(clicks, const Duration(seconds: 1))) {
print("Клик №$c принят");
}
}
```
### 15. Шагомер по акселерометру
```dart
import 'dart:math';
Stream<(double, double, double)> accelerometer() async* {
final rnd = Random();
while (true) {
await Future.delayed(const Duration(milliseconds: 100));
final spike = rnd.nextInt(4) == 0 ? 6.0 : 0.0; // имитация шага
yield (rnd.nextDouble(), rnd.nextDouble(), 9.8 + spike);
}
}
Stream<int> stepCounter(Stream<(double, double, double)> data, {double threshold = 12}) async* {
var steps = 0;
var above = false;
await for (final (x, y, z) in data) {
final magnitude = sqrt(x * x + y * y + z * z);
if (magnitude > threshold && !above) {
above = true;
yield ++steps;
} else if (magnitude <= threshold) {
above = false;
}
}
}
void main() async {
await for (final s in stepCounter(accelerometer()).take(5)) {
print("Шагов: $s");
}
}
```
### 16. Автопереподключение
```dart
import 'dart:async';
Stream<String> chatStream() async* {
for (var i = 1; i <= 3; i++) {
await Future.delayed(const Duration(milliseconds: 400));
yield "сообщение $i";
}
throw Exception("Обрыв соединения");
}
void listenWithReconnect(Stream<String> Function() factory, {int maxReconnects = 2}) {
var reconnects = 0;
void connect() {
factory().listen(
(m) => print("Чат: $m"),
onError: (e) {
print("Ошибка: $e");
if (reconnects++ < maxReconnects) {
print("Переподключение через 2 с...");
Timer(const Duration(seconds: 2), connect);
}
},
cancelOnError: true,
);
}
connect();
}
void main() => listenWithReconnect(chatStream);
```
### 17. Буфер: 5 событий или 3 секунды
```dart
import 'dart:async';
Stream<List<T>> buffer<T>(Stream<T> source, int count, Duration maxWait) {
final c = StreamController<List<T>>();
final buf = <T>[];
Timer? timer;
void flush() {
timer?.cancel();
timer = null;
if (buf.isNotEmpty) {
c.add(List<T>.from(buf));
buf.clear();
}
}
source.listen(
(e) {
buf.add(e);
timer ??= Timer(maxWait, flush);
if (buf.length >= count) flush();
},
onDone: () {
flush();
c.close();
},
);
return c.stream;
}
void main() async {
final clicks = Stream.periodic(const Duration(milliseconds: 700), (i) => "клик $i").take(8);
await for (final pack in buffer(clicks, 5, const Duration(seconds: 3))) {
print("Пачка: $pack");
}
}
```
### 18. OTP-таймер
```dart
Stream<int> otpCountdown([int seconds = 60]) =>
Stream.periodic(const Duration(seconds: 1), (i) => seconds - 1 - i).take(seconds);
void main() async {
await for (final left in otpCountdown(5)) { // для проверки — 5 секунд, в приложении 60
print("Код действителен ещё $left с");
}
print("Срок жизни кода истёк — запросите новый");
}
```
### 19. Скользящее среднее (окно 5)
```dart
Stream<double> movingAverage(Stream<num> source, {int window = 5}) async* {
final last = <num>[];
await for (final v in source) {
last.add(v);
if (last.length > window) last.removeAt(0);
yield last.reduce((a, b) => a + b) / last.length;
}
}
void main() async {
final data = Stream.fromIterable([10, 20, 30, 40, 50, 60, 70]);
await for (final avg in movingAverage(data)) {
print(avg.toStringAsFixed(1));
}
}
```
### 20. stream.last
```dart
void main() async {
final stream = Stream.fromIterable([3, 8, 15, 42]);
final last = await stream.last; // Stream -> Future
print("Последнее значение: $last"); // 42
}
```
### Блок 3: Шины событий и StreamController
### 21. EventBus (синглтон)
```dart
import 'dart:async';
class EventBus {
static final EventBus _instance = EventBus._();
factory EventBus() => _instance;
EventBus._();
final _controller = StreamController<Object>.broadcast();
void fire(Object event) => _controller.add(event);
Stream<T> on<T>() => _controller.stream.where((e) => e is T).cast<T>();
}
class LoggedIn {
final String name;
LoggedIn(this.name);
}
void main() async {
EventBus().on<LoggedIn>().listen((e) => print("Экран B: вошёл ${e.name}"));
EventBus().fire(LoggedIn("Аня")); // отправка с «экрана A»
await Future.delayed(const Duration(milliseconds: 50));
}
```
### 22. CartStore
```dart
import 'dart:async';
class CartState {
final int count;
final double total;
CartState(this.count, this.total);
@override
String toString() => "Товаров: $count, сумма: $total";
}
class CartStore {
final Map<String, double> _items = {};
final _controller = StreamController<CartState>.broadcast();
Stream<CartState> get stream => _controller.stream;
void _emit() => _controller.add(CartState(_items.length, _items.values.fold(0.0, (a, b) => a + b)));
void add(String name, double price) {
_items[name] = price;
_emit();
}
void remove(String name) {
_items.remove(name);
_emit();
}
void dispose() => _controller.close();
}
void main() async {
final s = CartStore();
s.stream.listen(print);
s.add("Кофе", 250);
s.add("Булка", 90);
s.remove("Кофе");
await Future.delayed(const Duration(milliseconds: 50));
s.dispose();
}
```
### 23. Бейдж непрочитанных
```dart
import 'dart:async';
class UnreadBadgeController {
int _count = 0;
final _controller = StreamController<int>.broadcast();
Stream<int> get stream => _controller.stream;
void onNewMessage() => _controller.add(++_count);
void markAllRead() {
_count = 0;
_controller.add(0);
}
void dispose() => _controller.close();
}
void main() async {
final b = UnreadBadgeController();
b.stream.listen((n) => print("Бейдж: $n"));
b.onNewMessage();
b.onNewMessage();
b.markAllRead();
await Future.delayed(const Duration(milliseconds: 50));
b.dispose();
}
```
### 24. AudioPlayerBloc
```dart
import 'dart:async';
sealed class PlayerState {}
class PlayerStopped extends PlayerState {}
class PlayerPlaying extends PlayerState {
final String track;
PlayerPlaying(this.track);
}
class PlayerPaused extends PlayerState {
final String track;
PlayerPaused(this.track);
}
class AudioPlayerBloc {
PlayerState _state = PlayerStopped();
final _controller = StreamController<PlayerState>.broadcast();
Stream<PlayerState> get states => _controller.stream;
void _set(PlayerState s) {
_state = s;
_controller.add(s);
}
void play(String track) => _set(PlayerPlaying(track));
void pause() {
if (_state case PlayerPlaying(:final track)) _set(PlayerPaused(track));
}
void resume() {
if (_state case PlayerPaused(:final track)) _set(PlayerPlaying(track));
}
void stop() => _set(PlayerStopped());
void dispose() => _controller.close();
}
String label(PlayerState s) => switch (s) {
PlayerStopped() => "Остановлено",
PlayerPlaying(:final track) => "Играет: $track",
PlayerPaused(:final track) => "Пауза: $track",
};
void main() async {
final b = AudioPlayerBloc();
b.states.listen((s) => print(label(s)));
b.play("Track 1");
b.pause();
b.resume();
b.stop();
await Future.delayed(const Duration(milliseconds: 50));
b.dispose();
}
```
### 25. Pagination Stream
```dart
import 'dart:async';
sealed class FeedEvent {}
class LoadNextPage extends FeedEvent {}
class FeedBloc {
final _events = StreamController<FeedEvent>();
final _news = StreamController<List<String>>.broadcast();
final List<String> _all = [];
int _page = 0;
FeedBloc() {
_events.stream.listen((e) async {
if (e is LoadNextPage) {
await Future.delayed(const Duration(milliseconds: 300)); // «сеть»
_all.addAll(List.generate(5, (i) => "Новость ${_page * 5 + i + 1}"));
_page++;
_news.add(List.unmodifiable(_all));
}
});
}
Sink<FeedEvent> get input => _events.sink;
Stream<List<String>> get news => _news.stream;
void dispose() {
_events.close();
_news.close();
}
}
void main() async {
final f = FeedBloc();
f.news.listen((n) => print("В ленте ${n.length}: последняя — ${n.last}"));
f.input.add(LoadNextPage());
f.input.add(LoadNextPage());
await Future.delayed(const Duration(seconds: 1));
f.dispose();
}
```
### 26. Реактивный переключатель темы
```dart
import 'dart:async';
enum ThemeMode { light, dark }
class ThemeManager {
ThemeMode _current = ThemeMode.light;
final _controller = StreamController<ThemeMode>.broadcast();
// BehaviorSubject-подобие: новый подписчик сразу получает текущее значение
Stream<ThemeMode> get stream async* {
yield _current;
yield* _controller.stream;
}
void toggle() {
_current = _current == ThemeMode.light ? ThemeMode.dark : ThemeMode.light;
_controller.add(_current);
}
void dispose() => _controller.close();
}
void main() async {
final m = ThemeManager();
m.stream.listen((t) => print("Тема: ${t.name}"));
await Future.delayed(const Duration(milliseconds: 20));
m.toggle();
m.toggle();
await Future.delayed(const Duration(milliseconds: 20));
m.dispose();
}
```
### 27. Connectivity Stream
```dart
import 'dart:async';
enum Connection { wifi, cellular, none }
class ConnectivityManager {
final _controller = StreamController<Connection>.broadcast();
Connection current = Connection.none;
Stream<Connection> get stream => _controller.stream;
void update(Connection c) {
if (c == current) return;
current = c;
_controller.add(c);
}
void dispose() => _controller.close();
}
void main() async {
final m = ConnectivityManager();
m.stream.listen((c) => print("Модуль кэша: ${c.name}"));
m.stream.listen((c) => print("Модуль чата: ${c.name}"));
m.update(Connection.wifi);
m.update(Connection.wifi); // дубликат игнорируется
m.update(Connection.none);
await Future.delayed(const Duration(milliseconds: 50));
m.dispose();
}
```
### 28. Auto-dispose контроллер
```dart
import 'dart:async';
class AutoDisposeController<T> {
late final StreamController<T> _c;
AutoDisposeController() {
_c = StreamController<T>.broadcast(onCancel: () {
if (!_c.hasListener) {
print("Подписчиков не осталось — закрываем контроллер");
_c.close();
}
});
}
bool get isClosed => _c.isClosed;
Stream<T> get stream => _c.stream;
void add(T v) {
if (!_c.isClosed) _c.add(v);
}
}
void main() async {
final c = AutoDisposeController<int>();
final s1 = c.stream.listen(print);
final s2 = c.stream.listen(print);
c.add(1);
await Future.delayed(const Duration(milliseconds: 20));
await s1.cancel();
await s2.cancel(); // здесь закроется
print("Закрыт: ${c.isClosed}");
}
```
### 29. Валидация формы (combineLatest вручную)
```dart
import 'dart:async';
class RegistrationBloc {
final _email = StreamController<String>.broadcast();
final _password = StreamController<String>.broadcast();
final _submit = StreamController<bool>.broadcast();
String? _lastEmail, _lastPass;
RegistrationBloc() {
_email.stream.listen((v) {
_lastEmail = v;
_check();
});
_password.stream.listen((v) {
_lastPass = v;
_check();
});
}
void _check() {
if (_lastEmail == null || _lastPass == null) return; // combineLatest ждёт оба потока
final ok = RegExp(r"^[\w.+-]+@[\w-]+\.[\w.]+$").hasMatch(_lastEmail!) && _lastPass!.length >= 8;
_submit.add(ok);
}
Sink<String> get emailIn => _email.sink;
Sink<String> get passwordIn => _password.sink;
Stream<bool> get isSubmitEnabled => _submit.stream;
void dispose() {
_email.close();
_password.close();
_submit.close();
}
}
void main() async {
final b = RegistrationBloc();
b.isSubmitEnabled.listen((v) => print("Кнопка активна: $v"));
b.emailIn.add("a@b.ru");
b.passwordIn.add("123");
b.passwordIn.add("12345678");
await Future.delayed(const Duration(milliseconds: 50));
b.dispose();
}
```
### 30. Логгер крашей
```dart
import 'dart:async';
class CrashReport {
final Object error;
final StackTrace stack;
CrashReport(this.error, this.stack);
}
final _crashes = StreamController<CrashReport>.broadcast();
void startCrashUploader() => _crashes.stream.listen((r) => print("Отправка в аналитику: ${r.error}"));
void main() {
startCrashUploader();
runZonedGuarded(() {
Future.delayed(const Duration(milliseconds: 100), () => throw StateError("Неотловленная ошибка"));
}, (error, stack) => _crashes.add(CrashReport(error, stack)));
}
```
### Блок 4: Изоляты
### 31. Миниатюры
```dart
import 'dart:isolate';
import 'dart:typed_data';
// RGBA-изображение w×h -> уменьшение в 2 раза (усреднение 2×2)
Uint8List downscale(Uint8List src, int w, int h) {
final nw = w ~/ 2, nh = h ~/ 2;
final out = Uint8List(nw * nh * 4);
for (var y = 0; y < nh; y++) {
for (var x = 0; x < nw; x++) {
for (var c = 0; c < 4; c++) {
var sum = 0;
for (var dy = 0; dy < 2; dy++) {
for (var dx = 0; dx < 2; dx++) {
sum += src[((y * 2 + dy) * w + (x * 2 + dx)) * 4 + c];
}
}
out[(y * nw + x) * 4 + c] = sum ~/ 4;
}
}
}
return out;
}
void main() async {
const w = 2000, h = 2000;
final image = Uint8List(w * h * 4)..fillRange(0, w * h * 4, 128);
final thumb = await Isolate.run(() => downscale(image, w, h));
print("${image.length} -> ${thumb.length} байт");
}
```
### 32. Шифрование большого текста
```dart
import 'dart:convert';
import 'dart:isolate';
String xorEncrypt(String text, String key) {
final k = utf8.encode(key);
final data = utf8.encode(text);
return base64.encode([for (var i = 0; i < data.length; i++) data[i] ^ k[i % k.length]]);
}
void main() async {
final text = List.generate(500000, (i) => "Заметка номер $i").join("\n");
final encrypted = await Isolate.run(() => xorEncrypt(text, "secret"));
print("Зашифровано: ${encrypted.length} символов");
}
```
### 33. Поиск по 100 000 документов
```dart
import 'dart:isolate';
List<String> searchTop10(String query, List<String> docs) {
final q = query.toLowerCase();
final scored = <(int, String)>[];
for (final d in docs) {
final text = d.toLowerCase();
var score = 0;
var idx = text.indexOf(q);
while (idx != -1) {
score++;
idx = text.indexOf(q, idx + q.length);
}
if (score > 0) scored.add((score, d));
}
scored.sort((a, b) => b.$1.compareTo(a.$1));
return scored.take(10).map((e) => e.$2).toList();
}
void main() async {
final docs = List.generate(100000, (i) => "Документ $i: ${i % 7 == 0 ? 'flutter flutter dart' : i % 3 == 0 ? 'flutter' : 'текст'}");
final top = await Isolate.run(() => searchTop10("flutter", docs));
top.forEach(print);
}
```
### 34. GeoJSON в фоне
```dart
import 'dart:convert';
import 'dart:isolate';
List<List<List<double>>> parsePolygons(String geoJson) {
final data = jsonDecode(geoJson) as Map<String, dynamic>;
return [
for (final f in data["features"] as List)
for (final ring in (f["geometry"]["coordinates"] as List))
[for (final p in ring as List) [(p[0] as num).toDouble(), (p[1] as num).toDouble()]]
];
}
void main() async {
final geo = jsonEncode({
"type": "FeatureCollection",
"features": List.generate(2000, (i) => {
"type": "Feature",
"geometry": {
"type": "Polygon",
"coordinates": [[[37.0 + i * 0.001, 55.0], [37.1 + i * 0.001, 55.0], [37.1 + i * 0.001, 55.1], [37.0 + i * 0.001, 55.0]]]
}
})
});
final polygons = await Isolate.run(() => parsePolygons(geo));
print("Районов: ${polygons.length}, вершин в первом: ${polygons.first.length}");
}
```
### 35. Сжатие логов (gzip)
```dart
import 'dart:convert';
import 'dart:io';
import 'dart:isolate';
List<int> compressLogs(List<String> lines) => gzip.encode(utf8.encode(lines.join("\n")));
void main() async {
final logs = List.generate(200000, (i) => "2026-11-07 12:00:00 INFO событие $i");
final raw = utf8.encode(logs.join("\n")).length;
final packed = await Isolate.run(() => compressLogs(logs));
print("Было $raw байт, стало ${packed.length} байт");
}
```
### 36. Worker Pool (2 изолята)
```dart
import 'dart:async';
import 'dart:isolate';
void _workerMain(SendPort mainPort) {
final port = ReceivePort();
mainPort.send(port.sendPort);
port.listen((msg) {
final (int id, int n) = msg as (int, int);
var sum = 0;
for (var i = 1; i <= n; i++) {
sum += i * i;
}
mainPort.send((id, sum));
});
}
class _Worker {
final Isolate isolate;
final SendPort send;
_Worker(this.isolate, this.send);
}
class WorkerPool {
final List<_Worker> _workers = [];
final Map<int, Completer<int>> _pending = {};
int _nextId = 0, _rr = 0;
final _results = ReceivePort();
Completer<SendPort>? _handshake;
Future<void> start(int size) async {
_results.listen((msg) {
if (msg is SendPort) {
_handshake!.complete(msg);
} else {
final (int id, int sum) = msg as (int, int);
_pending.remove(id)!.complete(sum);
}
});
for (var i = 0; i < size; i++) {
_handshake = Completer<SendPort>();
final isolate = await Isolate.spawn(_workerMain, _results.sendPort);
_workers.add(_Worker(isolate, await _handshake!.future));
}
}
Future<int> sumOfSquares(int n) {
final id = _nextId++;
final c = Completer<int>();
_pending[id] = c;
_workers[_rr++ % _workers.length].send.send((id, n));
return c.future;
}
void dispose() {
for (final w in _workers) {
w.isolate.kill(priority: Isolate.immediate);
}
_results.close();
}
}
void main() async {
final pool = WorkerPool();
await pool.start(2);
final results = await Future.wait([for (final n in [1000, 5000, 20000, 100]) pool.sumOfSquares(n)]);
print(results);
pool.dispose();
}
```
### 37. PDF-отчёт (текстовая структура)
```dart
import 'dart:isolate';
String buildReport(Map<String, double> monthlyRevenue) {
final sb = StringBuffer("ФИНАНСОВЫЙ ОТЧЁТ ЗА ГОД\n=======================\n");
var total = 0.0;
monthlyRevenue.forEach((m, v) {
sb.writeln("$m: ${v.toStringAsFixed(2)} ₽");
total += v;
});
sb.writeln("-----------------------\nИТОГО: ${total.toStringAsFixed(2)} ₽");
return sb.toString();
}
void main() async {
final data = {for (var m = 1; m <= 12; m++) "Месяц $m": m * 125000.5};
print(await Isolate.run(() => buildReport(data)));
}
```
### 38. Сортировка 1 000 000 чисел
```dart
import 'dart:isolate';
import 'dart:math';
void main() async {
final rnd = Random(1);
final data = List.generate(1000000, (_) => rnd.nextInt(1 << 30));
var sw = Stopwatch()..start();
final copy = List<int>.from(data)..sort();
print("В главном потоке: ${sw.elapsedMilliseconds} мс (UI заблокирован), min=${copy.first}");
sw = Stopwatch()..start();
final sorted = await Isolate.run(() => List<int>.from(data)..sort());
print("В изоляте: ${sw.elapsedMilliseconds} мс (UI свободен), min=${sorted.first}");
}
```
### 39. Конвертер аудио-семплов
```dart
import 'dart:isolate';
import 'dart:typed_data';
// 16-bit PCM -> float [-1.0, 1.0]
Float32List pcmToFloat(Int16List samples) {
final out = Float32List(samples.length);
for (var i = 0; i < samples.length; i++) {
out[i] = samples[i] / 32768.0;
}
return out;
}
void main() async {
final pcm = Int16List.fromList(List.generate(44100 * 10, (i) => (i % 200 - 100) * 300));
final floats = await Isolate.run(() => pcmToFloat(pcm));
print("Семплов: ${floats.length}, первый: ${floats.first}");
}
```
### 40. TransferableTypedData
```dart
import 'dart:isolate';
import 'dart:typed_data';
int checksum(TransferableTypedData t) {
final bytes = t.materialize().asUint8List(); // данные переданы без копирования
var sum = 0;
for (final b in bytes) {
sum = (sum + b) & 0xFFFFFFFF;
}
return sum;
}
void main() async {
final buffer = Uint8List(50 * 1024 * 1024)..fillRange(0, 50 * 1024 * 1024, 1);
final transferable = TransferableTypedData.fromList([buffer]);
final sum = await Isolate.run(() => checksum(transferable));
print("Контрольная сумма: $sum");
}
```
### Блок 5: Сложные асинхронные архитектуры
### 41. CancelToken
```dart
import 'dart:async';
class CancellationException implements Exception {
@override
String toString() => "CancellationException: операция отменена пользователем";
}
class CancelToken {
bool _cancelled = false;
final _completer = Completer<void>();
bool get isCancelled => _cancelled;
Future<void> get whenCancelled => _completer.future;
void cancel() {
if (_cancelled) return;
_cancelled = true;
_completer.complete();
}
void throwIfCancelled() {
if (_cancelled) throw CancellationException();
}
}
Future<String> download(CancelToken token) async {
for (var i = 1; i <= 5; i++) {
await Future.any([Future.delayed(const Duration(seconds: 1)), token.whenCancelled]);
token.throwIfCancelled();
print("Скачано ${i * 20}%");
}
return "Готово";
}
void main() async {
final token = CancelToken();
Future.delayed(const Duration(milliseconds: 2500), token.cancel); // пользователь ушёл со страницы
try {
print(await download(token));
} on CancellationException catch (e) {
print(e);
}
}
```
### 42. Репликация с разрешением конфликтов (Last-Write-Wins)
```dart
class Change {
final String key, value;
final DateTime timestamp;
Change(this.key, this.value, this.timestamp);
}
Future<Map<String, Change>> mergeQueues(Stream<Change> a, Stream<Change> b) async {
final result = <String, Change>{};
void apply(Change c) {
final existing = result[c.key];
if (existing == null || c.timestamp.isAfter(existing.timestamp)) result[c.key] = c;
}
await Future.wait([a.forEach(apply), b.forEach(apply)]);
return result;
}
Stream<Change> queue(List<Change> changes, int delayMs) async* {
for (final c in changes) {
await Future.delayed(Duration(milliseconds: delayMs));
yield c;
}
}
void main() async {
final t = DateTime(2026, 11, 7, 12);
final a = queue([Change("title", "A1", t), Change("note", "A-note", t.add(const Duration(minutes: 5)))], 100);
final b = queue([Change("title", "B1", t.add(const Duration(minutes: 1)))], 150);
final merged = await mergeQueues(a, b);
merged.forEach((k, c) => print("$k = ${c.value}")); // title = B1, note = A-note
}
```
### 43. Очередь офлайн-заказов
```dart
import 'dart:async';
class OfflineOrderQueue {
final List<String> _queue = [];
bool _online = false;
bool _sending = false;
Future<void> submit(String order) async {
_queue.add(order);
print("Заказ '$order' в очереди (${_queue.length})");
await _flush();
}
Future<void> setOnline(bool value) async {
_online = value;
await _flush();
}
Future<void> _flush() async {
if (!_online || _sending) return;
_sending = true;
while (_queue.isNotEmpty && _online) {
final order = _queue.first;
await Future.delayed(const Duration(milliseconds: 300)); // отправка
_queue.removeAt(0);
print("Отправлен: $order");
}
_sending = false;
}
}
void main() async {
final q = OfflineOrderQueue();
await q.submit("Пицца");
await q.submit("Суши");
await q.setOnline(true); // отправятся по одному
}
```
### 44. Circuit Breaker
```dart
class CircuitOpenException implements Exception {
@override
String toString() => "Цепь разорвана: сервис временно недоступен";
}
class CircuitBreaker {
final int threshold;
final Duration cooldown;
int _failures = 0;
DateTime? _openedAt;
CircuitBreaker({this.threshold = 5, this.cooldown = const Duration(seconds: 30)});
Future<T> call<T>(Future<T> Function() request) async {
if (_openedAt != null) {
if (DateTime.now().difference(_openedAt!) < cooldown) throw CircuitOpenException();
_openedAt = null; // полуоткрытое состояние: пробуем снова
_failures = 0;
}
try {
final result = await request();
_failures = 0;
return result;
} catch (e) {
if (++_failures >= threshold) {
_openedAt = DateTime.now();
print("5 ошибок подряд — режим «Разрыв цепи» на ${cooldown.inSeconds} с");
}
rethrow;
}
}
}
void main() async {
final cb = CircuitBreaker(cooldown: const Duration(seconds: 2));
for (var i = 1; i <= 8; i++) {
try {
await cb.call(() async => throw Exception("500"));
} catch (e) {
print("Запрос $i: $e");
}
}
}
```
### 45. Семафор (лимит 3)
```dart
import 'dart:async';
import 'dart:collection';
class AsyncSemaphore {
final int maxConcurrent;
int _running = 0;
final Queue<Completer<void>> _waiters = Queue();
AsyncSemaphore(this.maxConcurrent);
Future<void> acquire() async {
if (_running < maxConcurrent) {
_running++;
return;
}
final c = Completer<void>();
_waiters.add(c);
await c.future;
}
void release() {
if (_waiters.isNotEmpty) {
_waiters.removeFirst().complete(); // слот передаётся следующему
} else {
_running--;
}
}
Future<T> run<T>(Future<T> Function() task) async {
await acquire();
try {
return await task();
} finally {
release();
}
}
}
void main() async {
final sem = AsyncSemaphore(3);
var active = 0;
await Future.wait(List.generate(8, (i) => sem.run(() async {
print("Запрос $i старт (одновременно: ${++active})");
await Future.delayed(const Duration(milliseconds: 500));
active--;
})));
}
```
### 46. Гонка состояний при поиске
```dart
import 'dart:math';
class SearchController {
int _requestId = 0;
Future<List<String>> _api(String q) async {
await Future.delayed(Duration(milliseconds: 100 + Random().nextInt(900))); // непредсказуемая задержка
return ["результат для '$q'"];
}
Future<void> search(String query, void Function(List<String>) onResult) async {
final id = ++_requestId;
final result = await _api(query);
if (id != _requestId) {
print("Устаревший ответ на '$query' отброшен");
return;
}
onResult(result);
}
}
void main() async {
final c = SearchController();
c.search("fl", (r) => print("Показано: $r"));
c.search("flutter", (r) => print("Показано: $r"));
await Future.delayed(const Duration(seconds: 2));
}
```
### 47. Жизненный цикл и подписки
```dart
import 'dart:async';
import 'package:flutter/widgets.dart';
class SubscriptionManager with WidgetsBindingObserver {
final List<StreamSubscription<dynamic>> _subs = [];
SubscriptionManager() {
WidgetsBinding.instance.addObserver(this);
}
void add(StreamSubscription<dynamic> s) => _subs.add(s);
@override
void didChangeAppLifecycleState(AppLifecycleState state) {
if (state == AppLifecycleState.paused) {
for (final s in _subs) {
s.pause(); // приложение свернули
}
} else if (state == AppLifecycleState.resumed) {
for (final s in _subs) {
if (s.isPaused) s.resume();
}
}
}
Future<void> dispose() async {
WidgetsBinding.instance.removeObserver(this);
await Future.wait(_subs.map((s) => s.cancel()));
_subs.clear();
}
}
```
### 48. Конечный автомат синхронизации
```dart
import 'dart:async';
sealed class SyncState {}
class Idle extends SyncState {}
class Syncing extends SyncState {
final double progress;
Syncing(this.progress);
}
class SyncSuccess extends SyncState {}
class SyncError extends SyncState {
final String message;
SyncError(this.message);
}
class SyncMachine {
SyncState _state = Idle();
final _controller = StreamController<SyncState>.broadcast();
Stream<SyncState> get states => _controller.stream;
void _emit(SyncState s) {
_state = s;
_controller.add(s);
}
Future<void> start({bool fail = false}) async {
if (_state is Syncing) {
print("Синхронизация уже идёт — повторный запуск проигнорирован");
return;
}
try {
for (var i = 1; i <= 4; i++) {
_emit(Syncing(i / 4));
await Future.delayed(const Duration(milliseconds: 300));
if (fail && i == 3) throw Exception("Обрыв связи");
}
_emit(SyncSuccess());
} catch (e) {
_emit(SyncError("$e"));
}
}
void dispose() => _controller.close();
}
String label(SyncState s) => switch (s) {
Idle() => "Ожидание",
Syncing(:final progress) => "Синхронизация ${(progress * 100).round()}%",
SyncSuccess() => "Успех",
SyncError(:final message) => "Ошибка: $message",
};
void main() async {
final m = SyncMachine();
m.states.listen((s) => print(label(s)));
final f = m.start();
await m.start(); // отклонён
await f;
await m.start(fail: true);
m.dispose();
}
```
### 49. Многоуровневый кэш Memory → Disk → Network
```dart
class LayeredCache {
final Map<String, String> _memory = {};
final Map<String, String> _disk = {}; // имитация диска
Future<String> _network(String key) async {
await Future.delayed(const Duration(milliseconds: 800));
return "данные($key) из сети";
}
Future<String> get(String key) async {
if (_memory.containsKey(key)) {
print("[$key] из ОЗУ");
return _memory[key]!;
}
await Future.delayed(const Duration(milliseconds: 100)); // чтение с диска
if (_disk.containsKey(key)) {
print("[$key] с диска");
return _memory[key] = _disk[key]!;
}
print("[$key] из сети");
final value = await _network(key);
_disk[key] = value;
_memory[key] = value;
return value;
}
}
void main() async {
final c = LayeredCache();
await c.get("user"); // сеть
await c.get("user"); // ОЗУ
c._memory.clear();
await c.get("user"); // диск
}
```
### 50. Реактивный модуль финансового терминала
```dart
import 'dart:async';
import 'dart:isolate';
import 'dart:math';
// Имитатор WebSocket: котировки каждые 5 мс, иногда обрыв соединения
Stream<double> quotesSocket() async* {
final rnd = Random();
var price = 100.0;
while (true) {
await Future.delayed(const Duration(milliseconds: 5));
if (rnd.nextInt(300) == 0) throw Exception("WebSocket: соединение потеряно");
price += (rnd.nextDouble() - 0.5);
yield price;
}
}
// Волатильность = стандартное отклонение окна цен
double volatility(List<double> prices) {
final mean = prices.reduce((a, b) => a + b) / prices.length;
final variance = prices.map((p) => (p - mean) * (p - mean)).reduce((a, b) => a + b) / prices.length;
return sqrt(variance);
}
// Троттлинг до ~60 кадров/сек
Stream<T> throttle<T>(Stream<T> source, Duration interval) async* {
final sw = Stopwatch()..start();
var first = true;
await for (final e in source) {
if (first || sw.elapsed >= interval) {
first = false;
sw.reset();
yield e;
}
}
}
class TerminalModule {
final _output = StreamController<(double price, double volatility)>.broadcast();
final List<double> _window = [];
StreamSubscription<double>? _sub;
bool _disposed = false;
Stream<(double, double)> get updates => _output.stream;
void start() {
_sub?.cancel();
_sub = throttle(quotesSocket(), const Duration(milliseconds: 16)).listen(
(price) async {
_window.add(price);
if (_window.length > 50) _window.removeAt(0);
if (_window.length < 10) return;
final snapshot = List<double>.from(_window);
final vol = await Isolate.run(() => volatility(snapshot));
if (!_disposed) _output.add((price, vol));
},
onError: (e) {
print("Ошибка: $e. Переподключение через 1 с...");
Timer(const Duration(seconds: 1), () {
if (!_disposed) start();
});
},
cancelOnError: true,
);
}
Future<void> dispose() async {
_disposed = true;
await _sub?.cancel();
await _output.close();
}
}
void main() async {
final t = TerminalModule();
t.updates.listen((u) => print("Цена ${u.$1.toStringAsFixed(2)} | волатильность ${u.$2.toStringAsFixed(3)}"));
t.start();
await Future.delayed(const Duration(seconds: 3));
await t.dispose();
print("Модуль остановлен");
}
```
