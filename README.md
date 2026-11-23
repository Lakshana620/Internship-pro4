# Internship-pro4
Build a multi-screen converter app: a home menu and separate screens for length, weight and temperature.
import 'package:flutter/material.dart';

// ---------- 1. Units described as data ----------
// Each value is the unit's factor to the base unit (metre for length, kg for weight).
const lengthUnits = {
  'm': 1.0,
  'km': 1000.0,
  'mile': 1609.344,
  'cm': 0.01,
  'ft': 0.3048,
};

const weightUnits = {
  'g': 0.001,
  'kg': 1.0,
  'lb': 0.45359237,
  'oz': 0.028349523125,
};

const temperatureUnits = ['°C', '°F', 'K'];

// ---------- 2. Pure conversion functions (no widgets) ----------
// Convert to the base unit first, then from the base unit to the target.
double convert(double value, String from, String to, Map<String, double> units) {
  final base = value * units[from]!;
  return base / units[to]!;
}

// Temperature uses formulas, so everything goes through Celsius.
double convertTemperature(double value, String from, String to) {
  double celsius;
  switch (from) {
    case '°F':
      celsius = (value - 32) * 5 / 9;
      break;
    case 'K':
      celsius = value - 273.15;
      break;
    default:
      celsius = value;
  }
  switch (to) {
    case '°F':
      return celsius * 9 / 5 + 32;
    case 'K':
      return celsius + 273.15;
    default:
      return celsius;
  }
}

// Rounds to 4 decimals for display only, and drops trailing zeros.
String formatNumber(double n) {
  var s = n.toStringAsFixed(4);
  if (s.contains('.')) {
    s = s.replaceAll(RegExp(r'0+$'), '');
    s = s.replaceAll(RegExp(r'\.$'), '');
  }
  return (s == '-0') ? '0' : s;
}

// ---------- 3. One description per converter screen ----------
class ConverterType {
  final String title;
  final IconData icon;
  final List<String> units;
  final double Function(double value, String from, String to) convertFn;

  ConverterType({
    required this.title,
    required this.icon,
    required this.units,
    required this.convertFn,
  });
}

final converters = <ConverterType>[
  ConverterType(
    title: 'Length',
    icon: Icons.straighten,
    units: lengthUnits.keys.toList(),
    convertFn: (v, f, t) => convert(v, f, t, lengthUnits),
  ),
  ConverterType(
    title: 'Weight',
    icon: Icons.monitor_weight_outlined,
    units: weightUnits.keys.toList(),
    convertFn: (v, f, t) => convert(v, f, t, weightUnits),
  ),
  ConverterType(
    title: 'Temperature',
    icon: Icons.thermostat,
    units: temperatureUnits,
    convertFn: convertTemperature,
  ),
];

// ---------- 4. Tests as checks (see the DartPad console) ----------
bool close(double a, double b) => (a - b).abs() < 0.0001;

void check(String name, bool ok) => print('${ok ? 'PASS' : 'FAIL'}: $name');

void runChecks() {
  check('1 km = 1000 m', close(convert(1, 'km', 'm', lengthUnits), 1000));
  check('1 mile = 1609.344 m', close(convert(1, 'mile', 'm', lengthUnits), 1609.344));
  check('1 kg = 1000 g', close(convert(1, 'kg', 'g', weightUnits), 1000));
  check('1 lb = 0.45359237 kg', close(convert(1, 'lb', 'kg', weightUnits), 0.45359237));
  check('100 °C = 212 °F', close(convertTemperature(100, '°C', '°F'), 212));
  check('0 °C = 273.15 K', close(convertTemperature(0, '°C', 'K'), 273.15));
  check('-40 °C = -40 °F', close(convertTemperature(-40, '°C', '°F'), -40));
  check('212 °F = 100 °C', close(convertTemperature(212, '°F', '°C'), 100));
}

void main() {
  runChecks();
  assert(close(convert(1, 'km', 'm', lengthUnits), 1000));
  assert(close(convertTemperature(100, '°C', '°F'), 212));
  runApp(MaterialApp(
    debugShowCheckedModeBanner: false,
    theme: ThemeData(colorSchemeSeed: Colors.teal, useMaterial3: true),
    home: const HomePage(),
  ));
}

// ---------- 5. Home screen ----------
class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        backgroundColor: Colors.teal,
        foregroundColor: Colors.white,
        title: const Text('Converters'),
      ),
      body: ListView(
        padding: const EdgeInsets.all(12),
        children: converters.map((type) {
          return Card(
            child: ListTile(
              leading: Icon(type.icon, color: Colors.teal),
              title: Text(type.title),
              trailing: const Icon(Icons.chevron_right),
              onTap: () => Navigator.push(
                context,
                MaterialPageRoute(builder: (_) => ConverterPage(type: type)),
              ),
            ),
          );
        }).toList(),
      ),
    );
  }
}

// ---------- 6. One reusable converter screen ----------
class ConverterPage extends StatefulWidget {
  final ConverterType type;
  const ConverterPage({super.key, required this.type});

  @override
  State<ConverterPage> createState() => _ConverterPageState();
}

class _ConverterPageState extends State<ConverterPage> {
  final controller = TextEditingController();
  late String from;
  late String to;

  @override
  void initState() {
    super.initState();
    from = widget.type.units[0];
    to = widget.type.units[1];
  }

  @override
  void dispose() {
    controller.dispose();
    super.dispose();
  }

  void swapUnits() {
    setState(() {
      final temp = from;
      from = to;
      to = temp;
    });
  }

  Widget unitDropdown(String value, ValueChanged<String> onChanged) {
    return DropdownButton<String>(
      value: value,
      isExpanded: true,
      items: widget.type.units
          .map((u) => DropdownMenuItem(value: u, child: Text(u)))
          .toList(),
      onChanged: (v) => onChanged(v!),
    );
  }

  @override
  Widget build(BuildContext context) {
    final text = controller.text.trim();
    final value = double.tryParse(text);

    String message;
    bool isError = false;
    if (text.isEmpty) {
      message = 'Enter a number to convert';
    } else if (value == null) {
      message = '"$text" is not a valid number';
      isError = true;
    } else {
      final result = widget.type.convertFn(value, from, to);
      message = '${formatNumber(value)} $from = ${formatNumber(result)} $to';
    }

    return Scaffold(
      appBar: AppBar(
        backgroundColor: Colors.teal,
        foregroundColor: Colors.white,
        title: Text('${widget.type.title} converter'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            TextField(
              controller: controller,
              onChanged: (_) => setState(() {}),
              keyboardType: const TextInputType.numberWithOptions(
                  decimal: true, signed: true),
              decoration: const InputDecoration(
                labelText: 'Value',
                border: OutlineInputBorder(),
              ),
            ),
            const SizedBox(height: 16),
            Row(
              children: [
                Expanded(child: unitDropdown(from, (v) => setState(() => from = v))),
                IconButton(
                  tooltip: 'Swap units',
                  onPressed: swapUnits,
                  icon: const Icon(Icons.swap_horiz),
                ),
                Expanded(child: unitDropdown(to, (v) => setState(() => to = v))),
              ],
            ),
            const SizedBox(height: 24),
            Card(
              child: Padding(
                padding: const EdgeInsets.all(20),
                child: Text(
                  message,
                  textAlign: TextAlign.center,
                  style: TextStyle(
                    fontSize: 20,
                    color: isError ? Theme.of(context).colorScheme.error : null,
                  ),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
