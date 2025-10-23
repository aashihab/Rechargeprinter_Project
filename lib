import 'dart:convert';

import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';
import 'package:blue_thermal_printer/blue_thermal_printer.dart';

void main() {
  runApp(const MaterialApp(home: DescoReceiptApp()));
}

class DescoReceiptApp extends StatefulWidget {
  const DescoReceiptApp({super.key});

  @override
  State<DescoReceiptApp> createState() => _DescoReceiptAppState();
}

class _DescoReceiptAppState extends State<DescoReceiptApp> {
  final _accountController = TextEditingController();
  late final WebViewController _webViewController;
  BlueThermalPrinter bluetooth = BlueThermalPrinter.instance;
  List<BluetoothDevice> _devices = [];
  BluetoothDevice? _selectedDevice;
  List<Map<String, String>> _receipts = [];

  @override
  void initState() {
    super.initState();
    _webViewController = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted)
      ..loadRequest(Uri.parse('https://prepaid.desco.org.bd/customer/#/customer-login'));
    _initBluetooth();
  }

  void _initBluetooth() async {
    try {
      _devices = await bluetooth.getBondedDevices();
      setState(() {});
    } catch (e) {
      print('Error initializing Bluetooth: $e');
    }
  }

  void _fetchReceipts() async {
    final accountNumber = _accountController.text;
    if (accountNumber.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Please enter an account number')),
      );
      return;
    }

    await _webViewController.runJavaScript(
      'document.querySelector("input[placeholder=\\\"Account/Meter No\\\"]").value = "$accountNumber";'
    );
    await _webViewController.runJavaScript('document.querySelector("button").click();');

    // Wait for the page to load
    await Future.delayed(const Duration(seconds: 5));

    final receiptsData = await _webViewController.runJavaScriptReturningResult(
      '(() => {'
      '  const rows = Array.from(document.querySelectorAll(".table tbody tr"));'
      '  const receipts = rows.slice(0, 5).map(row => {'
      '    const columns = row.querySelectorAll("td");'
      '    return {'
      '      date: columns[1].innerText,'
      '      amount: columns[2].innerText,'
      '      receipt: columns[3].innerText,'
      '    };'
      '  });'
      '  return JSON.stringify(receipts);'
      '})();'
    );

    if (receiptsData != null) {
      final List<dynamic> decodedReceipts = jsonDecode(receiptsData.toString());
      setState(() {
        _receipts = decodedReceipts.map((r) => Map<String, String>.from(r)).toList();
      });
    }
  }

  void _printReceipt(Map<String, String> receipt) async {
    if (_selectedDevice == null) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Please select a printer')),
      );
      return;
    }

    try {
      await bluetooth.connect(_selectedDevice!);
      await bluetooth.printCustom("DESCO PREPAID RECEIPT", 1, 1);
      await bluetooth.printCustom("--------------------------------", 1, 1);
      await bluetooth.printCustom("Date: ${receipt['date']}", 1, 1);
      await bluetooth.printCustom("Amount: ${receipt['amount']}", 1, 1);
      await bluetooth.printCustom("Receipt: ${receipt['receipt']}", 1, 1);
      await bluetooth.printCustom("--------------------------------", 1, 1);
      await bluetooth.paperCut();
      await bluetooth.disconnect();
    } catch (e) {
      print('Error printing: $e');
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('DESCO Receipt Printer')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            TextField(
              controller: _accountController,
              decoration: const InputDecoration(labelText: 'Account/Meter No'),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: _fetchReceipts,
              child: const Text('Fetch Receipts'),
            ),
            const SizedBox(height: 16),
            DropdownButton<BluetoothDevice>(
              hint: const Text('Select Printer'),
              value: _selectedDevice,
              items: _devices.map((device) {
                return DropdownMenuItem(
                  value: device,
                  child: Text(device.name ?? 'Unknown Device'),
                );
              }).toList(),
              onChanged: (device) {
                setState(() {
                  _selectedDevice = device;
                });
              },
            ),
            const SizedBox(height: 16),
            Expanded(
              child: _receipts.isEmpty
                  ? WebViewWidget(controller: _webViewController)
                  : ListView.builder(
                      itemCount: _receipts.length,
                      itemBuilder: (context, index) {
                        final receipt = _receipts[index];
                        return ListTile(
                          title: Text('Date: ${receipt['date']}'),
                          subtitle: Text('Amount: ${receipt['amount']}'),
                          trailing: IconButton(
                            icon: const Icon(Icons.print),
                            onPressed: () => _printReceipt(receipt),
                          ),
                        );
                      },
                    ),
            ),
          ],
        ),
      ),
    );
  }
}

