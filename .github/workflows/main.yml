import 'dart:convert';

import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:intl/intl.dart';
import 'package:path/path.dart' as p;
import 'package:sqflite/sqflite.dart';
import 'package:url_launcher/url_launcher.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await LocalStore.instance.database;
  runApp(const BlackFridayWatchApp());
}

const _providers = <String, String>{
  'digikala': 'دیجی‌کالا',
  'torob': 'ترب',
  'emalls': 'ایمالز',
  'technolife': 'تکنولایف',
};

class PriceResult {
  final String provider;
  final String title;
  final int priceToman;
  final String url;
  final String? seller;
  final String note;
  const PriceResult({required this.provider, required this.title, required this.priceToman, required this.url, this.seller, this.note = ''});
  Map<String, Object?> toMap(int productId) => {
    'product_id': productId, 'provider': provider, 'title': title,
    'price_toman': priceToman, 'url': url, 'seller': seller,
    'checked_at': DateTime.now().toIso8601String(),
  };
}

class LocalStore {
  LocalStore._();
  static final instance = LocalStore._();
  Database? _db;
  Future<Database> get database async {
    if (_db != null) return _db!;
    final dir = await getDatabasesPath();
    _db = await openDatabase(p.join(dir, 'black_friday_watch.db'), version: 1,
      onCreate: (db, _) async {
        await db.execute('''CREATE TABLE products(id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, source TEXT NOT NULL, created_at TEXT NOT NULL)''');
        await db.execute('''CREATE TABLE prices(id INTEGER PRIMARY KEY AUTOINCREMENT, product_id INTEGER NOT NULL, provider TEXT NOT NULL, title TEXT NOT NULL, price_toman INTEGER NOT NULL, url TEXT NOT NULL, seller TEXT, checked_at TEXT NOT NULL, FOREIGN KEY(product_id) REFERENCES products(id) ON DELETE CASCADE)''');
        await db.execute('CREATE INDEX prices_product_date ON prices(product_id, checked_at)');
      },
    );
    await _db!.execute('PRAGMA foreign_keys = ON');
    return _db!;
  }
  Future<List<Map<String, Object?>>> products() async => (await database).query('products', orderBy: 'created_at DESC');
  Future<int> addProduct(String name, String source) async => (await database).insert('products', {'name': name, 'source': source, 'created_at': DateTime.now().toIso8601String()});
  Future<void> savePrices(int id, List<PriceResult> results) async {
    final db = await database;
    final batch = db.batch();
    for (final r in results) { batch.insert('prices', r.toMap(id)); }
    await batch.commit(noResult: true);
  }
  Future<List<Map<String, Object?>>> history(int id) async => (await database).query('prices', where: 'product_id=?', whereArgs: [id], orderBy: 'checked_at DESC');
  Future<void> deleteProduct(int id) async {
    final db = await database;
    await db.delete('prices', where: 'product_id=?', whereArgs: [id]);
    await db.delete('products', where: 'id=?', whereArgs: [id]);
  }
  Future<List<Map<String, Object?>>> latest(int id) async {
    final rows = await history(id);
    final seen = <String>{};
    return rows.where((r) => seen.add(r['provider'] as String)).toList();
  }
}

class PriceFinder {
  static final _client = http.Client();
  static const _headers = {
    'User-Agent': 'Mozilla/5.0 (Linux; Android 13) AppleWebKit/537.36 Chrome/124.0 Mobile Safari/537.36',
    'Accept': 'text/html,application/json,application/ld+json,*/*',
    'Accept-Language': 'fa-IR,fa;q=0.9,en;q=0.7',
  };

  static Future<Map<String, List<PriceResult>>> search(String input) async {
    final uri = Uri.tryParse(input.trim());
    final isUrl = uri != null && (uri.scheme == 'http' || uri.scheme == 'https') && uri.host.contains('.');
    String query = input.trim();
    final results = <String, List<PriceResult>>{};
    final errors = <String, String>{};

    if (isUrl) {
      final host = uri!.host.toLowerCase();
      final provider = _providerForHost(host);
      if (provider == null) {
        throw Exception('لینک باید متعلق به یکی از چهار سایت پشتیبانی‌شده باشد.');
      }
      try {
        final page = await _get(uri);
        final parsed = _parseHtml(provider, page, uri.toString());
        if (parsed.isNotEmpty) {
          results[provider] = parsed;
          query = parsed.first.title;
        } else {
          query = _meta(page, 'og:title') ?? _meta(page, 'twitter:title') ?? _htmlTitle(page) ?? _titleFromUrl(uri);
          query = _clean(query);
        }
      } catch (e) { errors[provider] = _friendlyError(e); query = _titleFromUrl(uri); }
    }

    final tasks = <Future<void>>[];
    for (final provider in _providers.keys) {
      if (results.containsKey(provider)) continue;
      tasks.add(() async {
        try {
          final found = await _searchProvider(provider, query);
          results[provider] = found;
          if (found.isEmpty) errors[provider] = 'نتیجه قابل استخراج نبود؛ ممکن است سایت ساختار صفحه را تغییر داده یا دسترسی را محدود کرده باشد.';
        } catch (e) { results[provider] = []; errors[provider] = _friendlyError(e); }
      }());
    }
    await Future.wait(tasks);
    // The original URL's provider may have returned no structured data; keep it visible as an attempted source.
    for (final key in _providers.keys) { results.putIfAbsent(key, () => []); }
    _lastErrors = errors;
    return results;
  }

  static Map<String, String> _lastErrors = {};
  static Map<String, String> get lastErrors => _lastErrors;

  static String? _providerForHost(String host) {
    if (host.contains('digikala.')) return 'digikala';
    if (host.contains('torob.')) return 'torob';
    if (host.contains('emalls.')) return 'emalls';
    if (host.contains('technolife.')) return 'technolife';
    return null;
  }

  static String _titleFromUrl(Uri uri) {
    final bits = uri.pathSegments.where((e) => e.isNotEmpty).toList();
    if (bits.isEmpty) return uri.host;
    return bits.last.replaceAll(RegExp(r'[-_]+'), ' ').replaceAll(RegExp(r'\.(html|aspx)$'), '');
  }

  static Future<String> _get(Uri uri) async {
    final response = await _client.get(uri, headers: _headers).timeout(const Duration(seconds: 18));
    if (response.statusCode < 200 || response.statusCode >= 400) throw Exception('پاسخ سایت: ${response.statusCode}');
    return utf8.decode(response.bodyBytes, allowMalformed: true);
  }

  static Future<List<PriceResult>> _searchProvider(String provider, String query) async {
    final encoded = Uri.encodeQueryComponent(query);
    if (provider == 'digikala') {
      final uri = Uri.parse('https://api.digikala.com/v1/search/?q=$encoded');
      final response = await _client.get(uri, headers: _headers).timeout(const Duration(seconds: 18));
      if (response.statusCode != 200) throw Exception('پاسخ دیجی‌کالا: ${response.statusCode}');
      final json = jsonDecode(utf8.decode(response.bodyBytes, allowMalformed: true));
      return _parseDigikala(json);
    }
    final urls = <String>{
      'torob': 'https://torob.com/search/?query=$encoded',
      'emalls': 'https://emalls.ir/Search/Products?SearchText=$encoded',
      'technolife': 'https://www.technolife.ir/search?query=$encoded',
    };
    final uri = Uri.parse(urls[provider]!);
    return _parseHtml(provider, await _get(uri), uri.toString());
  }

  static List<PriceResult> _parseDigikala(dynamic root) {
    final out = <PriceResult>[];
    void walk(dynamic node) {
      if (node is Map) {
        final map = Map<String, dynamic>.from(node);
        final nested = map['product'] is Map ? Map<String, dynamic>.from(map['product']) : map;
        final title = _firstString(nested, ['title_fa', 'title', 'name', 'product_title']);
        final id = nested['id'] ?? nested['product_id'];
        final priceObj = nested['price'];
        int? price;
        if (priceObj is Map) {
          price = _asInt(priceObj['selling_price'] ?? priceObj['price_toman'] ?? priceObj['final_price']);
          if (price != null && priceObj['price_toman'] == null) price = (price / 10).round(); // Digikala API price fields are commonly IRR.
        } else {
          price = _asInt(nested['price_toman'] ?? nested['selling_price_toman']);
        }
        final url = _firstString(nested, ['url', 'product_url']) ?? (id != null ? 'https://www.digikala.com/product/dkp-$id/' : '');
        if (title != null && price != null && price > 0 && url.isNotEmpty && out.length < 12) {
          out.add(PriceResult(provider: 'digikala', title: title, priceToman: price, url: url, seller: _seller(nested)));
        }
        for (final value in map.values) { if (value is Map || value is List) walk(value); }
      } else if (node is List) { for (final value in node) { if (value is Map || value is List) walk(value); } }
    }
    walk(root);
    final unique = <String, PriceResult>{};
    for (final item in out) { unique.putIfAbsent(item.url, () => item); }
    return unique.values.toList();
  }

  static String? _firstString(Map<String, dynamic> m, List<String> keys) {
    for (final k in keys) { final v = m[k]; if (v is String && v.trim().isNotEmpty) return v.trim(); }
    return null;
  }
  static String? _seller(Map<String, dynamic> m) {
    final s = m['default_seller'] ?? m['seller'];
    if (s is Map) return (s['title'] ?? s['name'])?.toString();
    return s is String ? s : null;
  }
  static int? _asInt(dynamic v) {
    if (v is num) return v.round();
    if (v is String) return int.tryParse(v.replaceAll(RegExp(r'[^0-9]'), ''));
    return null;
  }

  static List<PriceResult> _parseHtml(String provider, String html, String baseUrl) {
    final out = <PriceResult>[];
    final scripts = RegExp(r'''<script[^>]*(?:type=["']application/ld\+json["']|id=["']__NEXT_DATA__["']|type=["']application/json["'])[^>]*>(.*?)</script>''', caseSensitive: false, dotAll: true).allMatches(html);
    for (final match in scripts) {
      final raw = match.group(1)?.trim() ?? '';
      if (raw.isEmpty || raw.length > 2500000) continue;
      try {
        final decoded = jsonDecode(raw);
        _walkStructured(provider, decoded, baseUrl, out);
      } catch (_) {}
    }
    if (out.isEmpty) {
      final title = _meta(html, 'og:title') ?? _meta(html, 'twitter:title') ?? _htmlTitle(html);
      final price = _meta(html, 'product:price:amount') ?? _meta(html, 'og:price:amount') ?? _meta(html, 'twitter:data1');
      final amount = _asInt(price);
      if (title != null && amount != null && amount > 0) {
        out.add(PriceResult(provider: provider, title: _clean(title), priceToman: _normalizePrice(amount), url: _meta(html, 'og:url') ?? baseUrl));
      }
    }
    final unique = <String, PriceResult>{};
    for (final r in out) { unique.putIfAbsent(r.url, () => r); }
    return unique.values.take(15).toList();
  }

  static void _walkStructured(String provider, dynamic node, String base, List<PriceResult> out) {
    if (node is List) { for (final n in node) { _walkStructured(provider, n, base, out); } return; }
    if (node is! Map) return;
    final m = Map<String, dynamic>.from(node);
    final type = (m['@type'] ?? '').toString().toLowerCase();
    final name = (m['name'] ?? m['title_fa'] ?? m['title'] ?? m['headline'])?.toString();
    dynamic offer = m['offers'];
    if (offer is List && offer.isNotEmpty) offer = offer.first;
    int? price;
    String? seller;
    String url = (m['url'] ?? m['product_url'] ?? m['link'] ?? base).toString();
    if (offer is Map) {
      price = _asInt(offer['price'] ?? offer['lowPrice'] ?? offer['priceSpecification']?['price']);
      url = (m['url'] ?? offer['url'] ?? url).toString();
      seller = (offer['seller'] is Map ? offer['seller']['name'] : offer['seller'])?.toString();
    } else {
      price = _asInt(m['price_toman'] ?? m['price'] ?? m['selling_price_toman'] ?? m['final_price']);
    }
    if ((type.contains('product') || m.containsKey('offers') || (name != null && price != null)) && name != null && price != null && price > 0 && out.length < 60) {
      out.add(PriceResult(provider: provider, title: _clean(name), priceToman: _normalizePrice(price), url: _absolute(url, base), seller: seller));
    }
    for (final value in m.values) { if (value is Map || value is List) _walkStructured(provider, value, base, out); }
  }

  static int _normalizePrice(int price) => price;
  static String _absolute(String url, String base) {
    final u = Uri.tryParse(url);
    if (u == null) return base;
    return u.hasScheme ? url : Uri.parse(base).resolveUri(u).toString();
  }
  static String? _meta(String html, String key) {
    final escaped = RegExp.escape(key);
    final patterns = [
      RegExp("<meta[^>]+property=[\"']$escaped[\"'][^>]+content=[\"']([^\"']+)", caseSensitive: false),
      RegExp("<meta[^>]+content=[\"']([^\"']+)[\"'][^>]+property=[\"']$escaped[\"']", caseSensitive: false),
      RegExp("<meta[^>]+name=[\"']$escaped[\"'][^>]+content=[\"']([^\"']+)", caseSensitive: false),
    ];
    for (final ptn in patterns) { final m = ptn.firstMatch(html); if (m != null) return _decodeEntities(m.group(1)!); }
    return null;
  }
  static String? _htmlTitle(String html) { final m = RegExp(r'<title[^>]*>(.*?)</title>', caseSensitive: false, dotAll: true).firstMatch(html); return m == null ? null : _decodeEntities(m.group(1)!); }
  static String _clean(String s) => _decodeEntities(s).replaceAll(RegExp(r'\s+'), ' ').trim();
  static String _decodeEntities(String s) => s.replaceAll('&amp;', '&').replaceAll('&quot;', '"').replaceAll('&#39;', "'").replaceAll('&lt;', '<').replaceAll('&gt;', '>').replaceAll('&#x2F;', '/');
  static String _friendlyError(Object e) {
    final s = e.toString();
    if (s.contains('SocketException') || s.contains('Failed host lookup')) return 'اتصال اینترنت یا دسترسی به سایت برقرار نشد.';
    if (s.contains('TimeoutException')) return 'پاسخ سایت بیش از حد طول کشید.';
    return s.replaceFirst('Exception: ', '');
  }
}

class BlackFridayWatchApp extends StatelessWidget {
  const BlackFridayWatchApp({super.key});
  @override
  Widget build(BuildContext context) => MaterialApp(
    debugShowCheckedModeBanner: false,
    title: 'Black Friday Watch',
    theme: ThemeData(colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xff4f46e5)), useMaterial3: true, scaffoldBackgroundColor: const Color(0xfff7f7fb)),
    home: const HomePage(),
  );
}

class HomePage extends StatefulWidget { const HomePage({super.key}); @override State<HomePage> createState() => _HomePageState(); }
class _HomePageState extends State<HomePage> {
  final _input = TextEditingController();
  List<Map<String, Object?>> _products = [];
  bool _busy = false;
  String _status = 'نام محصول یا لینک آن را وارد کن.';
  Map<String, List<PriceResult>> _results = {};
  Map<String, String> _errors = {};

  @override void initState() { super.initState(); _loadProducts(); }
  @override void dispose() { _input.dispose(); super.dispose(); }
  Future<void> _loadProducts() async { final rows = await LocalStore.instance.products(); if (mounted) setState(() => _products = rows); }
  Future<void> _check() async {
    final value = _input.text.trim();
    if (value.isEmpty || _busy) return;
    setState(() { _busy = true; _status = 'در حال بررسی چهار منبع…'; _results = {}; _errors = {}; });
    try {
      final data = await PriceFinder.search(value);
      final name = value.startsWith('http') ? (data.values.expand((e) => e).firstOrNull?.title ?? value) : value;
      final id = await LocalStore.instance.addProduct(name, value);
      final all = data.values.expand((e) => e).toList();
      if (all.isNotEmpty) await LocalStore.instance.savePrices(id, all);
      if (!mounted) return;
      setState(() { _results = data; _errors = PriceFinder.lastErrors; _status = all.isEmpty ? 'قیمتی استخراج نشد؛ توضیح هر سایت را پایین ببین.' : '${all.length} نتیجه دریافت و در گوشی ذخیره شد.'; });
      await _loadProducts();
    } catch (e) {
      if (mounted) setState(() => _status = e.toString().replaceFirst('Exception: ', ''));
    } finally { if (mounted) setState(() => _busy = false); }
  }
  Future<void> _showHistory(Map<String, Object?> product) async {
    final id = product['id'] as int;
    final rows = await LocalStore.instance.history(id);
    if (!mounted) return;
    showModalBottomSheet(context: context, isScrollControlled: true, builder: (_) => Directionality(textDirection: TextDirection.rtl, child: SafeArea(child: Padding(padding: const EdgeInsets.all(18), child: SizedBox(height: 430, child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
      Text('تاریخچه: ${product['name']}', style: Theme.of(context).textTheme.titleMedium), const SizedBox(height: 12),
      if (rows.isEmpty) const Text('هنوز سابقه قیمتی برای این محصول ثبت نشده است.'),
      Expanded(child: ListView.builder(itemCount: rows.length, itemBuilder: (_, i) { final r = rows[i]; return ListTile(dense: true, title: Text('${_providers[r['provider']] ?? r['provider']} · ${_money(r['price_toman'] as int)}'), subtitle: Text(DateFormat('yyyy/MM/dd  HH:mm').format(DateTime.parse(r['checked_at'] as String))), trailing: IconButton(icon: const Icon(Icons.open_in_new), onPressed: () => _open(r['url'] as String))); })),
    ]))))));
  }
  Future<void> _delete(int id) async { await LocalStore.instance.deleteProduct(id); await _loadProducts(); if (mounted) setState(() {  }); }
  Future<void> _open(String url) async { final uri = Uri.tryParse(url); if (uri != null) { await launchUrl(uri, mode: LaunchMode.externalApplication); } }

  @override Widget build(BuildContext context) => Directionality(textDirection: TextDirection.rtl, child: Scaffold(
    appBar: AppBar(title: const Text('Black Friday Watch'), centerTitle: true, actions: [IconButton(tooltip: 'راهنما', onPressed: _help, icon: const Icon(Icons.info_outline))]),
    body: SafeArea(child: ListView(padding: const EdgeInsets.fromLTRB(16, 8, 16, 24), children: [
      Card(elevation: 0, child: Padding(padding: const EdgeInsets.all(16), child: Column(crossAxisAlignment: CrossAxisAlignment.stretch, children: [
        Text('مقایسه قیمت در لحظه', style: Theme.of(context).textTheme.titleLarge?.copyWith(fontWeight: FontWeight.bold)), const SizedBox(height: 6),
        const Text('نام کالا یا لینک محصول را وارد کن. نتایج و تاریخچه فقط روی همین گوشی ذخیره می‌شوند.'), const SizedBox(height: 14),
        TextField(controller: _input, textDirection: TextDirection.auto, decoration: const InputDecoration(border: OutlineInputBorder(), hintText: 'مثلاً گوشی سامسونگ S24 یا لینک کالا', prefixIcon: Icon(Icons.search)), onSubmitted: (_) => _check()),
        const SizedBox(height: 10), FilledButton.icon(onPressed: _busy ? null : _check, icon: _busy ? const SizedBox(width: 18, height: 18, child: CircularProgressIndicator(strokeWidth: 2)) : const Icon(Icons.compare_arrows), label: Text(_busy ? 'در حال جست‌وجو…' : 'جست‌وجو و مقایسه قیمت')),
        const SizedBox(height: 8), Text(_status, style: TextStyle(color: _status.contains('نشد') ? Colors.deepOrange : Colors.black54)),
      ]))),
      const SizedBox(height: 14),
      if (_results.isNotEmpty) ...[
        Text('نتایج بررسی', style: Theme.of(context).textTheme.titleLarge), const SizedBox(height: 8),
        ..._providers.entries.map((entry) {
          final list = _results[entry.key] ?? [];
          final sorted = [...list]..sort((a,b) => a.priceToman.compareTo(b.priceToman));
          return Card(elevation: 0, child: Padding(padding: const EdgeInsets.all(12), child: Column(crossAxisAlignment: CrossAxisAlignment.stretch, children: [
            Row(children: [Expanded(child: Text(entry.value, style: const TextStyle(fontWeight: FontWeight.bold))), if (sorted.isEmpty) const Chip(label: Text('بدون نتیجه')) else Chip(label: Text('${sorted.length} مورد'))]),
            if (sorted.isEmpty) Text(_errors[entry.key] ?? 'نتیجه‌ای پیدا نشد.', style: const TextStyle(color: Colors.black54, fontSize: 12)),
            ...sorted.take(5).map((r) => ListTile(contentPadding: EdgeInsets.zero, title: Text(r.title, maxLines: 2, overflow: TextOverflow.ellipsis), subtitle: Text('${_money(r.priceToman)} تومان${r.seller == null ? '' : ' · ${r.seller}'}'), trailing: IconButton(icon: const Icon(Icons.open_in_new), onPressed: () => _open(r.url)))),
          ])));
        }),
        Builder(builder: (_) {
          final all = _results.values.expand((e) => e).toList()..sort((a,b) => a.priceToman.compareTo(b.priceToman));
          if (all.isEmpty) return const SizedBox.shrink();
          final best = all.first;
          return Card(color: const Color(0xffe8f5e9), elevation: 0, child: ListTile(leading: const Icon(Icons.emoji_events_outlined, color: Colors.green), title: const Text('کمترین قیمت پیدا‌شده'), subtitle: Text('${_providers[best.provider]} · ${best.title}', maxLines: 2, overflow: TextOverflow.ellipsis), trailing: Text('${_money(best.priceToman)} تومان', style: const TextStyle(fontWeight: FontWeight.bold))));
        }),
        const SizedBox(height: 8), const Text('توجه: نتایج فقط وقتی قابل مقایسه‌اند که مدل، ظرفیت، رنگ، گارانتی و وضعیت موجودی یکسان باشد. قیمت نهایی را در صفحه فروشنده بررسی کن.', style: TextStyle(fontSize: 12, color: Colors.black54)),
      ],
      const SizedBox(height: 18),
      Row(children: [Expanded(child: Text('محصولات ذخیره‌شده', style: Theme.of(context).textTheme.titleLarge)), Text('${_products.length} مورد', style: const TextStyle(color: Colors.black54))]),
      if (_products.isEmpty) const Padding(padding: EdgeInsets.all(20), child: Center(child: Text('هنوز محصولی ذخیره نکرده‌ای.'))),
      ..._products.map((product) => Card(elevation: 0, child: ListTile(
        leading: const CircleAvatar(child: Icon(Icons.shopping_bag_outlined)), title: Text(product['name'] as String, maxLines: 2, overflow: TextOverflow.ellipsis),
        subtitle: Text('ذخیره‌شده در گوشی · ${DateFormat('yyyy/MM/dd').format(DateTime.parse(product['created_at'] as String))}'),
        onTap: () => _showHistory(product), trailing: PopupMenuButton<String>(onSelected: (v) { if (v == 'history') _showHistory(product); if (v == 'delete') _confirmDelete(product['id'] as int); }, itemBuilder: (_) => const [PopupMenuItem(value: 'history', child: Text('تاریخچه قیمت')), PopupMenuItem(value: 'delete', child: Text('حذف محصول'))]),
      ))),
    ])),
  ));

  Future<void> _confirmDelete(int id) async { final yes = await showDialog<bool>(context: context, builder: (_) => Directionality(textDirection: TextDirection.rtl, child: AlertDialog(title: const Text('حذف محصول؟'), content: const Text('تمام تاریخچه قیمت این محصول هم حذف می‌شود.'), actions: [TextButton(onPressed: () => Navigator.pop(context, false), child: const Text('انصراف')), FilledButton(onPressed: () => Navigator.pop(context, true), child: const Text('حذف'))]))); if (yes == true) await _delete(id); }
  void _help() { showAboutDialog(context: context, applicationName: 'Black Friday Watch', applicationVersion: '1.0.0', children: const [Text('این برنامه نتایج عمومی صفحات وب را در لحظه بررسی می‌کند و سوابق را در حافظه گوشی نگه می‌دارد. دسترسی سایت‌ها ممکن است تغییر کند یا محدود شود؛ در این حالت لینک مستقیم هر فروشگاه برای بررسی دستی در دسترس است. هیچ قیمت یا تخفیفی بدون دریافت داده واقعی نمایش داده نمی‌شود.')]); }
}

String _money(int n) => NumberFormat('#,###', 'en_US').format(n);

extension FirstOrNull<E> on Iterable<E> { E? get firstOrNull => isEmpty ? null : first; }
