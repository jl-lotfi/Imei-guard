import 'dart:convert';
import 'dart:math';

import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:http/http.dart' as http;
import 'package:shared_preferences/shared_preferences.dart';

// ===== models.dart =====
class Device {
  const Device({
    required this.imei,
    this.model = '',
    required this.status,
    this.city = '',
    this.date = '',
    this.note = '',
    required this.ownerId,
    this.example = false,
    required this.updatedAt,
  });

  final String imei;
  final String model;

  /// 'stolen' أو 'registered'
  final String status;
  final String city;

  /// تاريخ السرقة بصيغة yyyy-MM-dd
  final String date;
  final String note;
  final String ownerId;
  final bool example;
  final int updatedAt;

  bool get stolen => status == 'stolen';

  Map<String, dynamic> toJson() => {
        'imei': imei,
        'model': model,
        'status': status,
        'city': city,
        'date': date,
        'note': note,
        'owner_id': ownerId,
        'example': example,
        'updated_at': updatedAt,
      };

  factory Device.fromJson(Map<String, dynamic> j) => Device(
        imei: (j['imei'] ?? '').toString(),
        model: (j['model'] ?? '').toString(),
        status: (j['status'] ?? 'registered').toString(),
        city: (j['city'] ?? '').toString(),
        date: (j['date'] ?? '').toString(),
        note: (j['note'] ?? '').toString(),
        ownerId: (j['owner_id'] ?? '').toString(),
        example: j['example'] == true,
        updatedAt: (j['updated_at'] as num?)?.toInt() ?? 0,
      );
}

// ===== imei.dart =====
/// أدوات رقم IMEI: 15 رقماً، والرقم الأخير هو رقم تحقق بخوارزمية Luhn.
class Imei {
  static String digits(String v) {
    final d = v.replaceAll(RegExp(r'[^0-9]'), '');
    return d.length > 15 ? d.substring(0, 15) : d;
  }

  /// التنسيق المعتاد 2-6-6-1، فيكون TAC هو أول مجموعتين.
  static String format(String d) {
    final parts = <String>[
      d.substring(0, d.length < 2 ? d.length : 2),
      if (d.length > 2) d.substring(2, d.length < 8 ? d.length : 8),
      if (d.length > 8) d.substring(8, d.length < 14 ? d.length : 14),
      if (d.length > 14) d.substring(14),
    ];
    return parts.join(' ');
  }

  static bool luhn(String s) {
    var sum = 0;
    for (var i = 0; i < s.length; i++) {
      var d = int.parse(s[s.length - 1 - i]);
      if (i % 2 == 1) {
        d *= 2;
        if (d > 9) d -= 9;
      }
      sum += d;
    }
    return sum % 10 == 0;
  }
}

/// يجمّع الأرقام أثناء الكتابة.
class ImeiFormatter extends TextInputFormatter {
  @override
  TextEditingValue formatEditUpdate(
      TextEditingValue oldValue, TextEditingValue newValue) {
    final f = Imei.format(Imei.digits(newValue.text));
    return TextEditingValue(
      text: f,
      selection: TextSelection.collapsed(offset: f.length),
    );
  }
}

const _months = [
  'جانفي',
  'فيفري',
  'مارس',
  'أفريل',
  'ماي',
  'جوان',
  'جويلية',
  'أوت',
  'سبتمبر',
  'أكتوبر',
  'نوفمبر',
  'ديسمبر',
];

/// يحوّل 2026-11-02 إلى "2 نوفمبر 2026".
String fmtDate(String iso) {
  final p = iso.split('-');
  if (p.length != 3) return iso;
  final m = int.tryParse(p[1]);
  final d = int.tryParse(p[2]);
  if (m == null || d == null || m < 1 || m > 12) return iso;
  return '$d ${_months[m - 1]} ${p[0]}';
}

String isoDate(DateTime t) {
  final m = t.month.toString().padLeft(2, '0');
  final d = t.day.toString().padLeft(2, '0');
  return '${t.year}-$m-$d';
}

// ===== repository.dart =====
abstract class DeviceRepository {
  /// النص الذي يظهر في أعلى الشاشة لبيان مصدر البيانات.
  String get label;
  Future<Device?> get(String imei);
  Future<void> put(Device d);
  Future<void> delete(String imei);
  Future<List<Device>> mine(String ownerId);
}

/// تخزين على الهاتف نفسه. يعمل بلا إنترنت وبلا أي إعداد.
class LocalRepository implements DeviceRepository {
  LocalRepository(this._prefs) {
    if (!_prefs.containsKey(_key)) {
      final seed = <String, dynamic>{
        '490154203237518': const Device(
          imei: '490154203237518',
          model: 'Samsung Galaxy S23 (مثال)',
          status: 'stolen',
          city: 'تونس العاصمة',
          date: '2026-11-02',
          ownerId: 'demo',
          example: true,
          updatedAt: 1,
        ).toJson(),
        '352099001761481': const Device(
          imei: '352099001761481',
          model: 'iPhone 14 (مثال)',
          status: 'registered',
          ownerId: 'demo',
          example: true,
          updatedAt: 1,
        ).toJson(),
      };
      _write(seed);
    }
  }

  final SharedPreferences _prefs;
  static const _key = 'devices_v1';

  @override
  String get label => 'تجريبي على هذا الهاتف';

  Map<String, dynamic> _read() {
    final s = _prefs.getString(_key);
    if (s == null) return {};
    try {
      return Map<String, dynamic>.from(jsonDecode(s) as Map);
    } catch (_) {
      return {};
    }
  }

  Future<void> _write(Map<String, dynamic> m) =>
      _prefs.setString(_key, jsonEncode(m));

  @override
  Future<Device?> get(String imei) async {
    final m = _read()[imei];
    if (m == null) return null;
    return Device.fromJson(Map<String, dynamic>.from(m as Map));
  }

  @override
  Future<void> put(Device d) async {
    final m = _read();
    m[d.imei] = d.toJson();
    await _write(m);
  }

  @override
  Future<void> delete(String imei) async {
    final m = _read();
    m.remove(imei);
    await _write(m);
  }

  @override
  Future<List<Device>> mine(String ownerId) async {
    final list = _read()
        .values
        .map((v) => Device.fromJson(Map<String, dynamic>.from(v as Map)))
        .where((d) => d.ownerId == ownerId)
        .toList();
    list.sort((a, b) => b.updatedAt.compareTo(a.updatedAt));
    return list;
  }
}

/// قاعدة مشتركة على Supabase (جدول devices). انظر README.
class SupabaseRepository implements DeviceRepository {
  SupabaseRepository(String url, this._key)
      : _url = url.endsWith('/') ? url.substring(0, url.length - 1) : url;

  final String _url;
  final String _key;
  static const _timeout = Duration(seconds: 10);

  @override
  String get label => 'قاعدة مشتركة';

  Map<String, String> get _headers => {
        'apikey': _key,
        'Authorization': 'Bearer $_key',
        'Content-Type': 'application/json',
      };

  Uri _uri(String query) => Uri.parse('$_url/rest/v1/devices$query');

  List<Device> _parse(http.Response r) {
    if (r.statusCode != 200) {
      throw Exception('HTTP ${r.statusCode}');
    }
    final list = jsonDecode(utf8.decode(r.bodyBytes)) as List;
    return list
        .map((e) => Device.fromJson(Map<String, dynamic>.from(e as Map)))
        .toList();
  }

  @override
  Future<Device?> get(String imei) async {
    final r = await http
        .get(_uri('?imei=eq.$imei&select=*'), headers: _headers)
        .timeout(_timeout);
    final list = _parse(r);
    return list.isEmpty ? null : list.first;
  }

  @override
  Future<void> put(Device d) async {
    final r = await http
        .post(
          _uri(''),
          headers: {
            ..._headers,
            'Prefer': 'resolution=merge-duplicates,return=minimal',
          },
          body: jsonEncode(d.toJson()),
        )
        .timeout(_timeout);
    if (r.statusCode < 200 || r.statusCode >= 300) {
      throw Exception('HTTP ${r.statusCode}');
    }
  }

  @override
  Future<void> delete(String imei) async {
    final r = await http
        .delete(_uri('?imei=eq.$imei'), headers: _headers)
        .timeout(_timeout);
    if (r.statusCode < 200 || r.statusCode >= 300) {
      throw Exception('HTTP ${r.statusCode}');
    }
  }

  @override
  Future<List<Device>> mine(String ownerId) async {
    final r = await http
        .get(
          _uri('?owner_id=eq.$ownerId&select=*&order=updated_at.desc'),
          headers: _headers,
        )
        .timeout(_timeout);
    return _parse(r);
  }
}

// ===== widgets.dart =====
/// بطاقة بيضاء بحد رفيع، تحمل مجموعة حقول.
class Panel extends StatelessWidget {
  const Panel({super.key, required this.child});
  final Widget child;

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: cs.surface,
        border: Border.all(color: cs.outlineVariant),
        borderRadius: BorderRadius.circular(14),
      ),
      child: child,
    );
  }
}

/// حقل IMEI بخط أحادي المسافة وتجميع تلقائي للأرقام.
class ImeiField extends StatelessWidget {
  const ImeiField({
    super.key,
    required this.controller,
    this.label = 'رقم IMEI',
    this.onChanged,
    this.onSubmitted,
  });

  final TextEditingController controller;
  final String label;
  final ValueChanged<String>? onChanged;
  final ValueChanged<String>? onSubmitted;

  @override
  Widget build(BuildContext context) {
    return TextField(
      controller: controller,
      keyboardType: TextInputType.number,
      inputFormatters: [ImeiFormatter()],
      textDirection: TextDirection.ltr,
      textAlign: TextAlign.center,
      style: const TextStyle(
        fontFamily: 'monospace',
        fontSize: 22,
        letterSpacing: 1.2,
        fontWeight: FontWeight.w600,
      ),
      decoration: InputDecoration(
        labelText: label,
        hintText: '00 000000 000000 0',
        border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
      ),
      onChanged: onChanged,
      onSubmitted: onSubmitted,
    );
  }
}

// ===== screens/check_screen.dart =====
enum Kind { ok, bad, warn }

class CheckResult {
  const CheckResult(this.kind, this.title, this.body,
      {this.rows = const [], this.note});
  final Kind kind;
  final String title;
  final String body;
  final List<MapEntry<String, String>> rows;
  final String? note;
}

class CheckScreen extends StatefulWidget {
  const CheckScreen({super.key, required this.repo});
  final DeviceRepository repo;

  @override
  State<CheckScreen> createState() => _CheckScreenState();
}

class _CheckScreenState extends State<CheckScreen> {
  final _ctrl = TextEditingController();
  CheckResult? _result;
  bool _busy = false;

  static const _segColors = [
    Color(0xFF0F5C7A),
    Color(0xFF2E8B7A),
    Color(0xFFC27C0E),
  ];

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  Future<void> _run() async {
    final d = Imei.digits(_ctrl.text);
    if (d.length != 15) {
      setState(() {
        _result = CheckResult(Kind.warn, 'الرقم غير مكتمل',
            'أدخلت ${d.length} من 15 رقماً. اطلب *#06# لعرض الرقم كاملاً.');
      });
      return;
    }
    if (!Imei.luhn(d)) {
      setState(() {
        _result = const CheckResult(
            Kind.warn,
            'رقم IMEI غير صالح',
            'الرقم الأخير لا يطابق بقية الأرقام، أي أن رقماً كُتب بشكل خاطئ أو أن الرقم مزوَّر. أعد قراءته من الهاتف نفسه.');
      });
      return;
    }
    setState(() {
      _busy = true;
    });
    CheckResult result;
    try {
      final rec = await widget.repo.get(d);
      if (rec == null) {
        result = CheckResult(
          Kind.ok,
          'لا يوجد بلاغ سرقة',
          'هذا الرقم غير مبلَّغ عنه في التطبيق.',
          rows: [MapEntry('IMEI', Imei.format(d))],
          note:
              'غياب البلاغ لا يعني أن الهاتف غير مسروق. القاعدة تضم فقط ما سُجّل هنا، فاطلب فاتورة الشراء قبل الدفع.',
        );
      } else if (rec.stolen) {
        result = CheckResult(
          Kind.bad,
          'مبلَّغ عنه كمسروق',
          'لا تشترِ هذا الهاتف، وبلّغ الجهات الأمنية إن عُرض عليك.',
          rows: [
            MapEntry('الجهاز', rec.model.isEmpty ? 'غير مذكور' : rec.model),
            MapEntry('IMEI', Imei.format(d)),
            MapEntry('المدينة', rec.city.isEmpty ? 'غير مذكورة' : rec.city),
            MapEntry('تاريخ السرقة',
                rec.date.isEmpty ? 'غير مذكور' : fmtDate(rec.date)),
            if (rec.note.isNotEmpty) MapEntry('ملاحظة المالك', rec.note),
          ],
          note: rec.example ? 'هذا سجل تجريبي لعرض شكل النتيجة.' : null,
        );
      } else {
        result = CheckResult(
          Kind.ok,
          'جهاز مسجّل دون بلاغ',
          'سجّله مالكه لدينا ولا يوجد بلاغ سرقة عليه.',
          rows: [
            MapEntry('الجهاز', rec.model.isEmpty ? 'غير مذكور' : rec.model),
            MapEntry('IMEI', Imei.format(d)),
          ],
          note: rec.example ? 'هذا سجل تجريبي لعرض شكل النتيجة.' : null,
        );
      }
    } catch (_) {
      result = const CheckResult(Kind.warn, 'تعذّر الفحص',
          'لم نصل إلى القاعدة. تحقق من الاتصال وأعد المحاولة.');
    }
    if (!mounted) return;
    setState(() {
      _busy = false;
      _result = result;
    });
  }

  void _sample(String imei) {
    _ctrl.text = Imei.format(imei);
    setState(() {});
    _run();
  }

  Widget _cells(Color off, int len) {
    const sizes = [8, 6, 1];
    const offsets = [0, 8, 14];
    final cells = <Widget>[];
    for (var g = 0; g < 3; g++) {
      if (g > 0) cells.add(const SizedBox(width: 10));
      for (var i = 0; i < sizes[g]; i++) {
        final filled = offsets[g] + i < len;
        cells.add(Container(
          width: 14,
          height: 8,
          margin: const EdgeInsets.symmetric(horizontal: 1.5),
          decoration: BoxDecoration(
            color: filled ? _segColors[g] : off,
            borderRadius: BorderRadius.circular(2),
          ),
        ));
      }
    }
    return Directionality(
      textDirection: TextDirection.ltr,
      child: Row(mainAxisAlignment: MainAxisAlignment.center, children: cells),
    );
  }

  Widget _legendItem(Color c, String text, Color textColor) {
    return Row(mainAxisSize: MainAxisSize.min, children: [
      Container(
        width: 8,
        height: 8,
        decoration: BoxDecoration(color: c, borderRadius: BorderRadius.circular(2)),
      ),
      const SizedBox(width: 6),
      Text(text, style: TextStyle(fontSize: 12, color: textColor)),
    ]);
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    final len = Imei.digits(_ctrl.text).length;
    return ListView(
      padding: const EdgeInsets.all(16),
      children: [
        Panel(
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              ImeiField(
                controller: _ctrl,
                label: 'رقم IMEI للهاتف',
                onChanged: (_) => setState(() {}),
                onSubmitted: (_) => _run(),
              ),
              const SizedBox(height: 14),
              _cells(cs.outlineVariant, len),
              const SizedBox(height: 8),
              Wrap(
                alignment: WrapAlignment.center,
                spacing: 14,
                runSpacing: 4,
                children: [
                  _legendItem(_segColors[0], 'طراز الجهاز TAC (8)', cs.onSurfaceVariant),
                  _legendItem(_segColors[1], 'رقم تسلسلي (6)', cs.onSurfaceVariant),
                  _legendItem(_segColors[2], 'تحقق (1)', cs.onSurfaceVariant),
                ],
              ),
              const SizedBox(height: 12),
              Text(
                'لمعرفة الرقم اطلب *#06# من تطبيق الاتصال، أو اقرأه على علبة الهاتف.',
                style: TextStyle(color: cs.onSurfaceVariant, fontSize: 13),
              ),
              const SizedBox(height: 14),
              FilledButton(
                onPressed: _busy ? null : _run,
                style: FilledButton.styleFrom(minimumSize: const Size.fromHeight(50)),
                child: Text(_busy ? 'جارٍ الفحص...' : 'افحص الجهاز'),
              ),
            ],
          ),
        ),
        const SizedBox(height: 12),
        Wrap(
          spacing: 8,
          crossAxisAlignment: WrapCrossAlignment.center,
          children: [
            Text('جرّب مثالاً:', style: TextStyle(color: cs.onSurfaceVariant, fontSize: 13)),
            ActionChip(label: const Text('مبلَّغ عنه'), onPressed: () => _sample('490154203237518')),
            ActionChip(label: const Text('مسجّل'), onPressed: () => _sample('352099001761481')),
            ActionChip(label: const Text('بلا بلاغ'), onPressed: () => _sample('356938035643809')),
            ActionChip(label: const Text('رقم خاطئ'), onPressed: () => _sample('353918051234563')),
          ],
        ),
        if (_result != null) ...[
          const SizedBox(height: 16),
          _ResultCard(result: _result!),
        ],
      ],
    );
  }
}

class _ResultCard extends StatelessWidget {
  const _ResultCard({required this.result});
  final CheckResult result;

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    final dark = Theme.of(context).brightness == Brightness.dark;
    final Color bg;
    final Color fg;
    final IconData icon;
    switch (result.kind) {
      case Kind.ok:
        bg = dark ? const Color(0xFF11301F) : const Color(0xFFE1F3E9);
        fg = dark ? const Color(0xFF55D698) : const Color(0xFF126B42);
        icon = Icons.check_circle_outline;
        break;
      case Kind.bad:
        bg = dark ? const Color(0xFF3A1815) : const Color(0xFFFCE7E4);
        fg = dark ? const Color(0xFFFF9187) : const Color(0xFFA8231A);
        icon = Icons.report_problem_outlined;
        break;
      case Kind.warn:
        bg = dark ? const Color(0xFF33270B) : const Color(0xFFFFF0D1);
        fg = dark ? const Color(0xFFF2C063) : const Color(0xFF7D5200);
        icon = Icons.info_outline;
        break;
    }
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: bg,
        border: Border.all(color: fg),
        borderRadius: BorderRadius.circular(14),
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(children: [
            Icon(icon, color: fg, size: 30),
            const SizedBox(width: 10),
            Expanded(
              child: Text(result.title,
                  style: TextStyle(color: fg, fontSize: 20, fontWeight: FontWeight.w700)),
            ),
          ]),
          const SizedBox(height: 8),
          Text(result.body, style: TextStyle(color: cs.onSurface, fontSize: 14)),
          if (result.rows.isNotEmpty) ...[
            const SizedBox(height: 10),
            Divider(color: fg, height: 1),
            const SizedBox(height: 8),
            for (final r in result.rows)
              Padding(
                padding: const EdgeInsets.symmetric(vertical: 2),
                child: Row(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    SizedBox(
                      width: 96,
                      child: Text(r.key,
                          style: TextStyle(color: cs.onSurfaceVariant, fontSize: 14)),
                    ),
                    Expanded(
                      child: Text(r.value,
                          style: TextStyle(
                            color: cs.onSurface,
                            fontSize: 14,
                            fontFamily: r.key == 'IMEI' ? 'monospace' : null,
                          )),
                    ),
                  ],
                ),
              ),
          ],
          if (result.note != null) ...[
            const SizedBox(height: 10),
            Text(result.note!,
                style: TextStyle(color: cs.onSurfaceVariant, fontSize: 13)),
          ],
        ],
      ),
    );
  }
}

// ===== screens/report_screen.dart =====
/// قيم تُملأ مسبقاً عند الانتقال من شاشة «أجهزتي».
class Prefill {
  const Prefill({this.imei = '', this.model = '', this.status = 'stolen'});
  final String imei;
  final String model;
  final String status;
}

class ReportScreen extends StatefulWidget {
  const ReportScreen({
    super.key,
    required this.repo,
    required this.ownerId,
    required this.onSaved,
    this.initial,
  });

  final DeviceRepository repo;
  final String ownerId;
  final VoidCallback onSaved;
  final Prefill? initial;

  @override
  State<ReportScreen> createState() => _ReportScreenState();
}

class _ReportScreenState extends State<ReportScreen> {
  final _imei = TextEditingController();
  final _model = TextEditingController();
  final _city = TextEditingController();
  final _note = TextEditingController();
  String _status = 'stolen';
  DateTime _date = DateTime.now();
  String? _error;
  bool _busy = false;

  @override
  void initState() {
    super.initState();
    final p = widget.initial;
    if (p != null) {
      _imei.text = Imei.format(Imei.digits(p.imei));
      _model.text = p.model;
      _status = p.status;
    }
  }

  @override
  void dispose() {
    _imei.dispose();
    _model.dispose();
    _city.dispose();
    _note.dispose();
    super.dispose();
  }

  void _fail(String m) {
    setState(() {
      _error = m;
    });
  }

  Future<void> _pickDate() async {
    final picked = await showDatePicker(
      context: context,
      initialDate: _date,
      firstDate: DateTime(2010),
      lastDate: DateTime.now(),
      locale: const Locale('ar'),
    );
    if (picked != null) {
      setState(() {
        _date = picked;
      });
    }
  }

  Future<void> _submit() async {
    setState(() {
      _error = null;
    });
    final d = Imei.digits(_imei.text);
    final stolen = _status == 'stolen';
    if (d.length != 15) {
      _fail('رقم IMEI يجب أن يتكون من 15 رقماً.');
      return;
    }
    if (!Imei.luhn(d)) {
      _fail('رقم IMEI غير صالح. راجع الأرقام أو اطلب *#06# من الهاتف.');
      return;
    }
    if (stolen && _city.text.trim().isEmpty) {
      _fail('اكتب المدينة التي سُرق فيها الهاتف.');
      return;
    }
    setState(() {
      _busy = true;
    });
    try {
      final old = await widget.repo.get(d);
      if (old != null && old.ownerId != widget.ownerId) {
        _fail('هذا الرقم مسجّل باسم مستخدم آخر. إن كان هاتفك فعلاً فأثبت ملكيتك عبر الفاتورة (ميزة قادمة).');
      } else {
        await widget.repo.put(Device(
          imei: d,
          model: _model.text.trim(),
          status: _status,
          city: stolen ? _city.text.trim() : '',
          date: stolen ? isoDate(_date) : '',
          note: stolen ? _note.text.trim() : '',
          ownerId: widget.ownerId,
          updatedAt: DateTime.now().millisecondsSinceEpoch,
        ));
        if (!mounted) return;
        _imei.clear();
        _model.clear();
        _city.clear();
        _note.clear();
        _date = DateTime.now();
        ScaffoldMessenger.of(context).showSnackBar(SnackBar(
          content: Text(stolen ? 'تم تسجيل البلاغ' : 'تم تسجيل الجهاز'),
        ));
        widget.onSaved();
      }
    } catch (_) {
      _fail('تعذّر الحفظ. تحقق من الاتصال وأعد المحاولة.');
    }
    if (!mounted) return;
    setState(() {
      _busy = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    final stolen = _status == 'stolen';
    return ListView(
      padding: const EdgeInsets.all(16),
      children: [
        Panel(
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              ImeiField(controller: _imei),
              const SizedBox(height: 14),
              TextField(
                controller: _model,
                maxLength: 60,
                decoration: InputDecoration(
                  labelText: 'اسم الهاتف وطرازه',
                  hintText: 'مثال: Samsung Galaxy A54',
                  border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
                ),
              ),
              const SizedBox(height: 8),
              Text('ماذا تريد أن تفعل؟',
                  style: TextStyle(color: cs.onSurfaceVariant, fontSize: 13)),
              const SizedBox(height: 6),
              SegmentedButton<String>(
                segments: const [
                  ButtonSegment(value: 'stolen', label: Text('الإبلاغ عن سرقة')),
                  ButtonSegment(value: 'registered', label: Text('تسجيل فقط')),
                ],
                selected: {_status},
                onSelectionChanged: (s) {
                  setState(() {
                    _status = s.first;
                  });
                },
              ),
              if (stolen) ...[
                const SizedBox(height: 16),
                TextField(
                  controller: _city,
                  maxLength: 40,
                  decoration: InputDecoration(
                    labelText: 'المدينة التي سُرق فيها',
                    hintText: 'مثال: صفاقس',
                    border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
                  ),
                ),
                const SizedBox(height: 8),
                OutlinedButton.icon(
                  onPressed: _pickDate,
                  icon: const Icon(Icons.event),
                  label: Text('تاريخ السرقة: ${fmtDate(isoDate(_date))}'),
                  style: OutlinedButton.styleFrom(minimumSize: const Size.fromHeight(48)),
                ),
                const SizedBox(height: 14),
                TextField(
                  controller: _note,
                  maxLength: 200,
                  maxLines: 3,
                  decoration: InputDecoration(
                    labelText: 'ملاحظة (اختياري)',
                    hintText: 'مثال: غطاء أزرق وشاشة مشروخة من الزاوية',
                    border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
                  ),
                ),
              ],
              if (_error != null) ...[
                const SizedBox(height: 8),
                Text(_error!, style: TextStyle(color: cs.error, fontSize: 13)),
              ],
              const SizedBox(height: 12),
              FilledButton(
                onPressed: _busy ? null : _submit,
                style: FilledButton.styleFrom(minimumSize: const Size.fromHeight(50)),
                child: Text(_busy ? 'جارٍ الحفظ...' : 'حفظ'),
              ),
            ],
          ),
        ),
      ],
    );
  }
}

// ===== screens/mine_screen.dart =====
class MineScreen extends StatefulWidget {
  const MineScreen({
    super.key,
    required this.repo,
    required this.ownerId,
    required this.changes,
    required this.onReport,
  });

  final DeviceRepository repo;
  final String ownerId;
  final ValueNotifier<int> changes;
  final void Function(Prefill) onReport;

  @override
  State<MineScreen> createState() => _MineScreenState();
}

class _MineScreenState extends State<MineScreen> {
  late Future<List<Device>> _future;

  @override
  void initState() {
    super.initState();
    _future = widget.repo.mine(widget.ownerId);
    widget.changes.addListener(_reload);
  }

  @override
  void dispose() {
    widget.changes.removeListener(_reload);
    super.dispose();
  }

  void _reload() {
    if (!mounted) return;
    setState(() {
      _future = widget.repo.mine(widget.ownerId);
    });
  }

  void _snack(String m) {
    ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(m)));
  }

  Future<void> _found(Device d) async {
    try {
      await widget.repo.put(Device(
        imei: d.imei,
        model: d.model,
        status: 'registered',
        ownerId: d.ownerId,
        updatedAt: DateTime.now().millisecondsSinceEpoch,
      ));
      _snack('تم تحديث الحالة');
      widget.changes.value++;
    } catch (_) {
      _snack('تعذّر التحديث. تحقق من الاتصال.');
    }
  }

  Future<void> _remove(Device d) async {
    final ok = await showDialog<bool>(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('حذف الجهاز؟'),
        content: Text('سيُحذف السجل ${Imei.format(d.imei)} نهائياً.'),
        actions: [
          TextButton(onPressed: () => Navigator.pop(ctx, false), child: const Text('تراجع')),
          FilledButton(onPressed: () => Navigator.pop(ctx, true), child: const Text('حذف')),
        ],
      ),
    );
    if (ok != true) return;
    try {
      await widget.repo.delete(d.imei);
      _snack('تم الحذف');
      widget.changes.value++;
    } catch (_) {
      _snack('تعذّر الحذف. تحقق من الاتصال.');
    }
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    return FutureBuilder<List<Device>>(
      future: _future,
      builder: (context, snap) {
        if (snap.connectionState != ConnectionState.done) {
          return const Center(child: CircularProgressIndicator());
        }
        if (snap.hasError) {
          return Center(
            child: Column(mainAxisSize: MainAxisSize.min, children: [
              const Text('تعذّر الوصول إلى القاعدة الآن.'),
              const SizedBox(height: 12),
              OutlinedButton(onPressed: _reload, child: const Text('إعادة المحاولة')),
            ]),
          );
        }
        final list = snap.data ?? const <Device>[];
        if (list.isEmpty) {
          return ListView(
            padding: const EdgeInsets.all(16),
            children: [
              Panel(
                child: Column(children: [
                  const SizedBox(height: 12),
                  Text('لم تسجّل أي جهاز بعد',
                      style: Theme.of(context).textTheme.titleMedium),
                  const SizedBox(height: 8),
                  Text('سجّل هاتفك برقم IMEI حتى يسهل عليك الإبلاغ فوراً إن سُرق.',
                      textAlign: TextAlign.center,
                      style: TextStyle(color: cs.onSurfaceVariant)),
                  const SizedBox(height: 14),
                  FilledButton(
                    onPressed: () => widget.onReport(const Prefill(status: 'registered')),
                    child: const Text('إضافة جهاز'),
                  ),
                  const SizedBox(height: 12),
                ]),
              ),
            ],
          );
        }
        return RefreshIndicator(
          onRefresh: () async {
            _reload();
            await _future;
          },
          child: ListView(
            physics: const AlwaysScrollableScrollPhysics(),
            padding: const EdgeInsets.all(16),
            children: [
              for (final d in list) ...[
                _DeviceCard(
                  device: d,
                  onFound: () => _found(d),
                  onReportStolen: () => widget.onReport(
                      Prefill(imei: d.imei, model: d.model, status: 'stolen')),
                  onDelete: () => _remove(d),
                ),
                const SizedBox(height: 12),
              ],
              OutlinedButton.icon(
                onPressed: () => widget.onReport(const Prefill(status: 'registered')),
                icon: const Icon(Icons.add),
                label: const Text('إضافة جهاز'),
              ),
            ],
          ),
        );
      },
    );
  }
}

class _DeviceCard extends StatelessWidget {
  const _DeviceCard({
    required this.device,
    required this.onFound,
    required this.onReportStolen,
    required this.onDelete,
  });

  final Device device;
  final VoidCallback onFound;
  final VoidCallback onReportStolen;
  final VoidCallback onDelete;

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    final dark = Theme.of(context).brightness == Brightness.dark;
    final stolen = device.stolen;
    final pillBg = stolen
        ? (dark ? const Color(0xFF3A1815) : const Color(0xFFFCE7E4))
        : (dark ? const Color(0xFF11301F) : const Color(0xFFE1F3E9));
    final pillFg = stolen
        ? (dark ? const Color(0xFFFF9187) : const Color(0xFFA8231A))
        : (dark ? const Color(0xFF55D698) : const Color(0xFF126B42));
    final meta = [
      if (stolen && device.city.isNotEmpty) device.city,
      if (stolen && device.date.isNotEmpty) fmtDate(device.date),
    ].join(' · ');
    return Panel(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(device.model.isEmpty ? 'جهاز بدون اسم' : device.model,
                        style: const TextStyle(fontSize: 16, fontWeight: FontWeight.w600)),
                    Directionality(
                      textDirection: TextDirection.ltr,
                      child: Text(Imei.format(device.imei),
                          style: TextStyle(
                              fontFamily: 'monospace',
                              fontSize: 13,
                              color: cs.onSurfaceVariant)),
                    ),
                  ],
                ),
              ),
              Container(
                padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 2),
                decoration: BoxDecoration(
                  color: pillBg,
                  borderRadius: BorderRadius.circular(99),
                ),
                child: Text(stolen ? 'مبلَّغ عنه' : 'مسجّل',
                    style: TextStyle(color: pillFg, fontSize: 12, fontWeight: FontWeight.w600)),
              ),
            ],
          ),
          if (meta.isNotEmpty) ...[
            const SizedBox(height: 6),
            Text(meta, style: TextStyle(color: cs.onSurfaceVariant, fontSize: 13)),
          ],
          const SizedBox(height: 10),
          Wrap(
            spacing: 8,
            children: [
              if (stolen)
                OutlinedButton(onPressed: onFound, child: const Text('تم العثور عليه'))
              else
                OutlinedButton(onPressed: onReportStolen, child: const Text('أبلغ عن سرقته')),
              OutlinedButton(
                onPressed: onDelete,
                style: OutlinedButton.styleFrom(foregroundColor: cs.error),
                child: const Text('حذف'),
              ),
            ],
          ),
        ],
      ),
    );
  }
}

// ===== main.dart =====
const _supabaseUrl = String.fromEnvironment('SUPABASE_URL');
const _supabaseKey = String.fromEnvironment('SUPABASE_ANON_KEY');

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final prefs = await SharedPreferences.getInstance();

  var ownerId = prefs.getString('owner_id');
  if (ownerId == null) {
    final r = Random.secure();
    ownerId = List.generate(16, (_) => r.nextInt(16).toRadixString(16)).join();
    await prefs.setString('owner_id', ownerId);
  }

  final DeviceRepository repo =
      (_supabaseUrl.isNotEmpty && _supabaseKey.isNotEmpty)
          ? SupabaseRepository(_supabaseUrl, _supabaseKey)
          : LocalRepository(prefs);

  runApp(GuardApp(repo: repo, ownerId: ownerId));
}

class GuardApp extends StatelessWidget {
  const GuardApp({super.key, required this.repo, required this.ownerId});
  final DeviceRepository repo;
  final String ownerId;

  @override
  Widget build(BuildContext context) {
    const seed = Color(0xFF0F5C7A);
    return MaterialApp(
      title: 'حارس IMEI',
      debugShowCheckedModeBanner: false,
      locale: const Locale('ar'),
      supportedLocales: const [Locale('ar'), Locale('en')],
      localizationsDelegates: const [
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: seed),
        useMaterial3: true,
      ),
      darkTheme: ThemeData(
        colorScheme:
            ColorScheme.fromSeed(seedColor: seed, brightness: Brightness.dark),
        useMaterial3: true,
      ),
      themeMode: ThemeMode.system,
      home: Shell(repo: repo, ownerId: ownerId),
    );
  }
}

class Shell extends StatefulWidget {
  const Shell({super.key, required this.repo, required this.ownerId});
  final DeviceRepository repo;
  final String ownerId;

  @override
  State<Shell> createState() => _ShellState();
}

class _ShellState extends State<Shell> {
  int _tab = 0;
  final _changes = ValueNotifier<int>(0);
  Prefill? _prefill;
  int _prefillSeq = 0;

  @override
  void dispose() {
    _changes.dispose();
    super.dispose();
  }

  void _openReport(Prefill p) {
    setState(() {
      _prefill = p;
      _prefillSeq++;
      _tab = 1;
    });
  }

  void _saved() {
    _changes.value++;
    setState(() {
      _tab = 2;
    });
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    return Scaffold(
      appBar: AppBar(
        title: const Text('حارس IMEI', style: TextStyle(fontWeight: FontWeight.w700)),
        leading: Icon(Icons.shield_outlined, color: cs.primary),
        actions: [
          Padding(
            padding: const EdgeInsetsDirectional.only(end: 12),
            child: Chip(
              label: Text(widget.repo.label, style: const TextStyle(fontSize: 12)),
              visualDensity: VisualDensity.compact,
            ),
          ),
        ],
      ),
      body: IndexedStack(
        index: _tab,
        children: [
          CheckScreen(repo: widget.repo),
          ReportScreen(
            key: ValueKey(_prefillSeq),
            repo: widget.repo,
            ownerId: widget.ownerId,
            onSaved: _saved,
            initial: _prefill,
          ),
          MineScreen(
            repo: widget.repo,
            ownerId: widget.ownerId,
            changes: _changes,
            onReport: _openReport,
          ),
        ],
      ),
      bottomNavigationBar: NavigationBar(
        selectedIndex: _tab,
        onDestinationSelected: (i) {
          setState(() {
            _tab = i;
          });
        },
        destinations: const [
          NavigationDestination(icon: Icon(Icons.search), label: 'تحقق'),
          NavigationDestination(icon: Icon(Icons.outlined_flag), label: 'أبلغ'),
          NavigationDestination(icon: Icon(Icons.phone_android), label: 'أجهزتي'),
        ],
      ),
    );
  }
}
