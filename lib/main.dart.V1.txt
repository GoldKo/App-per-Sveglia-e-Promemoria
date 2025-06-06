import 'dart:convert'; // AGGIUNTO
import 'package:flutter/material.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/timezone.dart' as tz;
import 'package:timezone/data/latest.dart' as tz;
import 'package:shared_preferences/shared_preferences.dart';
import 'package:intl/intl.dart';
import 'dart:math';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await NotificationService().init();
  tz.initializeTimeZones();
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'SmartAlarm Pro',
      theme: ThemeData.dark(),
      home: const HomeScreen(),
      debugShowCheckedModeBanner: false,
    );
  }
}

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});

  @override
  _HomeScreenState createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  int _currentIndex = 0;

  final List<Widget> _screens = [
    const AlarmScreen(),
    const ReminderScreen(),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(_currentIndex == 0 ? 'Sveglie Intelligenti' : 'Promemoria'),
        centerTitle: true,
      ),
      body: _screens[_currentIndex],
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _currentIndex,
        items: const [
          BottomNavigationBarItem(
            icon: Icon(Icons.alarm),
            label: 'Sveglie',
          ),
          BottomNavigationBarItem(
            icon: Icon(Icons.notifications),
            label: 'Promemoria',
          ),
        ],
        onTap: (index) {
          setState(() => _currentIndex = index);
        },
      ),
    );
  }
}

class NotificationService {
  // SINGLETON
  static final NotificationService _instance = NotificationService._internal();
  factory NotificationService() => _instance;
  NotificationService._internal();

  final FlutterLocalNotificationsPlugin notificationsPlugin =
      FlutterLocalNotificationsPlugin();

  Future<void> init() async {
    const AndroidInitializationSettings androidSettings =
        AndroidInitializationSettings('app_icon');
    
    const DarwinInitializationSettings iosSettings =
        DarwinInitializationSettings(
      requestSoundPermission: false,
      requestBadgePermission: false,
      requestAlertPermission: true,
    );

    const InitializationSettings settings = InitializationSettings(
      android: androidSettings,
      iOS: iosSettings,
    );

    await notificationsPlugin.initialize(
      settings,
      onDidReceiveNotificationResponse: (NotificationResponse details) {},
    );
  }

  Future<void> scheduleAlarmNotification({
    required int id,
    required DateTime time,
    required String gameType,
    required String sound,
  }) async {
    await notificationsPlugin.zonedSchedule(
      id,
      'Sveglia!',
      'È ora di svegliarsi!',
      tz.TZDateTime.from(time, tz.local),
      NotificationDetails(
        android: AndroidNotificationDetails(
          'alarm_channel',
          'Sveglie',
          importance: Importance.max,
          priority: Priority.high,
          sound: RawResourceAndroidNotificationSound(sound),
          enableVibration: true,
          fullScreenIntent: true,
        ),
        iOS: DarwinNotificationDetails(
          sound: '$sound.caf',
          presentAlert: true,
          presentBadge: true,
          presentSound: true,
          interruptionLevel: InterruptionLevel.critical,
        ),
      ),
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      payload: gameType,
    );
  }

  Future<void> scheduleReminder({
    required int id,
    required String title,
    required String message,
    required DateTime startTime,
    required DateTime? endTime,
    required int interval,
  }) async {
    var currentTime = startTime;
    int notificationId = id;

    while (endTime == null || currentTime.isBefore(endTime)) {
      await notificationsPlugin.zonedSchedule(
        notificationId++,
        title,
        message,
        tz.TZDateTime.from(currentTime, tz.local),
        const NotificationDetails(
          android: AndroidNotificationDetails(
            'reminder_channel',
            'Promemoria',
            importance: Importance.high,
          ),
          iOS: DarwinNotificationDetails(
            presentAlert: true,
            presentBadge: true,
            sound: 'default',
          ),
        ),
        androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      );

      currentTime = currentTime.add(Duration(minutes: interval));
      if (endTime != null && currentTime.isAfter(endTime)) break;
    }
  }

  Future<void> cancelNotification(int id) async {
    await notificationsPlugin.cancel(id);
  }
}

class AlarmScreen extends StatefulWidget {
  const AlarmScreen({super.key});

  @override
  _AlarmScreenState createState() => _AlarmScreenState();
}

class _AlarmScreenState extends State<AlarmScreen> {
  List<Alarm> _alarms = [];
  final NotificationService _notificationService = NotificationService();
  final _soundOptions = ['beep', 'chime', 'ringtone'];
  final _gameOptions = ['Matematica', 'Fisica', 'Storia', 'Generale'];

  @override
  void initState() {
    super.initState();
    _loadAlarms();
  }

  void _loadAlarms() async {
    final prefs = await SharedPreferences.getInstance();
    List<String>? alarms = prefs.getStringList('alarms');
    
    if (alarms != null) {
      setState(() {
        _alarms = alarms.map((a) => Alarm.fromJson(a)).toList();
      });
    }
  }

  void _saveAlarms() async {
    final prefs = await SharedPreferences.getInstance();
    prefs.setStringList('alarms', _alarms.map((a) => a.toJson()).toList());
  }

  void _addAlarm(Alarm alarm) {
    setState(() {
      _alarms.add(alarm);
      _saveAlarms();
      _scheduleAlarm(alarm);
    });
  }

  void _scheduleAlarm(Alarm alarm) async {
    await _notificationService.scheduleAlarmNotification(
      id: alarm.id,
      time: alarm.time,
      gameType: alarm.gameType,
      sound: alarm.sound,
    );
  }

  void _deleteAlarm(int index) async {
    await _notificationService.cancelNotification(_alarms[index].id);
    setState(() {
      _alarms.removeAt(index);
      _saveAlarms();
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: _alarms.isEmpty
          ? const Center(child: Text('Nessuna sveglia impostata'))
          : ListView.builder(
              itemCount: _alarms.length,
              itemBuilder: (context, index) {
                Alarm alarm = _alarms[index];
                return ListTile(
                  title: Text(
                    '${alarm.time.hour}:${alarm.time.minute.toString().padLeft(2, '0')}',
                    style: const TextStyle(fontSize: 18),
                  ),
                  subtitle: Text('${alarm.gameType} (suono: ${alarm.sound})'),
                  trailing: IconButton(
                    icon: const Icon(Icons.delete, color: Colors.red),
                    onPressed: () => _deleteAlarm(index),
                  ),
                  onTap: () => _showAlarmGame(context, alarm),
                );
              },
            ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showAddAlarmDialog(context),
        child: const Icon(Icons.add),
      ),
    );
  }

  void _showAddAlarmDialog(BuildContext context) {
    TimeOfDay selectedTime = TimeOfDay.now();
    String selectedSound = _soundOptions.first;
    String selectedGame = _gameOptions.first;

    showDialog(
      context: context,
      builder: (context) {
        return StatefulBuilder(
          builder: (context, setStateDialog) {
            return AlertDialog(
              title: const Text('Nuova Sveglia'),
              content: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  ListTile(
                    title: Text('Orario: ${selectedTime.format(context)}'),
                    trailing: const Icon(Icons.access_time),
                    onTap: () async {
                      final time = await showTimePicker(
                        context: context,
                        initialTime: selectedTime,
                      );
                      if (time != null) {
                        setStateDialog(() {
                          selectedTime = time;
                        });
                      }
                    },
                  ),
                  const SizedBox(height: 10),
                  DropdownButtonFormField<String>(
                    value: selectedSound,
                    items: _soundOptions
                        .map((sound) => DropdownMenuItem(
                              value: sound,
                              child: Text(sound),
                            ))
                        .toList(),
                    onChanged: (value) {
                      if (value != null) setStateDialog(() => selectedSound = value);
                    },
                    decoration: const InputDecoration(
                      labelText: 'Suoneria',
                      border: OutlineInputBorder(),
                    ),
                  ),
                  const SizedBox(height: 10),
                  DropdownButtonFormField<String>(
                    value: selectedGame,
                    items: _gameOptions
                        .map((game) => DropdownMenuItem(
                              value: game,
                              child: Text(game),
                            ))
                        .toList(),
                    onChanged: (value) {
                      if (value != null) setStateDialog(() => selectedGame = value);
                    },
                    decoration: const InputDecoration(
                      labelText: 'Tipo di gioco',
                      border: OutlineInputBorder(),
                    ),
                  ),
                ],
              ),
              actions: [
                TextButton(
                  onPressed: () => Navigator.pop(context),
                  child: const Text('Annulla'),
                ),
                TextButton(
                  onPressed: () {
                    final now = DateTime.now();
                    var alarmTime = DateTime(
                      now.year,
                      now.month,
                      now.day,
                      selectedTime.hour,
                      selectedTime.minute,
                    );
                    if (alarmTime.isBefore(now)) {
                      alarmTime = alarmTime.add(const Duration(days: 1));
                    }

                    _addAlarm(Alarm(
                      id: DateTime.now().millisecondsSinceEpoch,
                      time: alarmTime,
                      sound: selectedSound,
                      gameType: selectedGame,
                    ));
                    Navigator.pop(context);
                  },
                  child: const Text('Salva'),
                ),
              ],
            );
          },
        );
      },
    );
  }

  void _showAlarmGame(BuildContext context, Alarm alarm) {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => AlarmGameScreen(alarm: alarm),
      ),
    );
  }
}

class Alarm {
  final int id;
  final DateTime time;
  final String sound;
  final String gameType;

  Alarm({
    required this.id,
    required this.time,
    required this.sound,
    required this.gameType,
  });

  String toJson() => jsonEncode({
    'id': id,
    'time': time.toIso8601String(),
    'sound': sound,
    'gameType': gameType,
  });

  factory Alarm.fromJson(String jsonStr) {
    final map = jsonDecode(jsonStr);
    return Alarm(
      id: map['id'],
      time: DateTime.parse(map['time']),
      sound: map['sound'],
      gameType: map['gameType'],
    );
  }
}

class AlarmGameScreen extends StatefulWidget {
  final Alarm alarm;

  const AlarmGameScreen({Key? key, required this.alarm}) : super(key: key);

  @override
  _AlarmGameScreenState createState() => _AlarmGameScreenState();
}

class _AlarmGameScreenState extends State<AlarmGameScreen> {
  late MathProblem _currentProblem;
  late String _userAnswer = '';
  final NotificationService _notificationService = NotificationService();
  final _random = Random();

  @override
  void initState() {
    super.initState();
    _generateProblem();
  }

  void _generateProblem() {
    setState(() {
      _userAnswer = '';
      _currentProblem = MathProblem.generate(widget.alarm.gameType, _random);
    });
  }

  void _checkAnswer() {
    if (_userAnswer.trim() == _currentProblem.answer) {
      _notificationService.cancelNotification(widget.alarm.id);
      Navigator.pop(context);
    } else {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Risposta errata! Riprova...')),
      );
      _generateProblem();
    }
  }

  void _posticipaSveglia() async {
    final newTime = DateTime.now().add(const Duration(minutes: 5));
    await _notificationService.scheduleAlarmNotification(
      id: widget.alarm.id,
      time: newTime,
      gameType: widget.alarm.gameType,
      sound: widget.alarm.sound,
    );
    Navigator.pop(context);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Disattiva Sveglia'),
      ),
      body: Center(
        child: Padding(
          padding: const EdgeInsets.all(30.0),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text(
                _currentProblem.question,
                style: const TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
                textAlign: TextAlign.center,
              ),
              const SizedBox(height: 20),
              TextField(
                decoration: const InputDecoration(
                  labelText: 'Risposta',
                  border: OutlineInputBorder(),
                ),
                style: const TextStyle(fontSize: 20),
                keyboardType: TextInputType.text,
                onChanged: (value) => _userAnswer = value,
                onSubmitted: (_) => _checkAnswer(),
              ),
              const SizedBox(height: 30),
              ElevatedButton(
                onPressed: _checkAnswer,
                child: const Text('Controlla risposta', style: TextStyle(fontSize: 18)),
                style: ElevatedButton.styleFrom(
                  padding: const EdgeInsets.symmetric(horizontal: 30, vertical: 15)
                ),
              ),
              const SizedBox(height: 10),
              TextButton(
                onPressed: _posticipaSveglia,
                child: const Text('Posticipa sveglia di 5 min', style: TextStyle(fontSize: 16)),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class MathProblem {
  final String question;
  final String answer;

  MathProblem({required this.question, required this.answer});

  factory MathProblem.generate(String gameType, Random random) {
    switch (gameType) {
      case 'Matematica':
        return _mathProblem(random);
      case 'Fisica':
        return _physicsProblem();
      case 'Storia':
        return _historyProblem();
      default:
        return _generalProblem();
    }
  }

  static MathProblem _mathProblem(Random random) {
    int num1 = random.nextInt(20) + 1;
    int num2 = random.nextInt(20) + 1;
    int num3 = random.nextInt(10) + 1;
    final operators = ['+', '-', '*', '/'];
    String operator = operators[random.nextInt(operators.length)];

    String expression;
    String answer;

    switch (operator) {
      case '+':
        expression = '$num1 + $num2';
        answer = (num1 + num2).toString();
        break;
      case '-':
        expression = '$num1 - $num2';
        answer = (num1 - num2).toString();
        break;
      case '*':
        expression = '$num1 × $num2';
        answer = (num1 * num2).toString();
        break;
      case '/':
        expression = '$num3 × $num2';
        answer = (num3 * num2).toString();
        break;
      default:
        expression = '$num1 + $num2';
        answer = (num1 + num2).toString();
    }

    return MathProblem(
      question: 'Calcola: $expression',
      answer: answer,
    );
  }

  static MathProblem _physicsProblem() {
    return MathProblem(
      question: 'Qual è la velocità della luce nel vuoto (in m/s)?',
      answer: '299792458',
    );
  }

  static MathProblem _historyProblem() {
    return MathProblem(
      question: "In che anno è caduto l'Impero Romano d'Occidente?",
      answer: '476',
    );
  }

  static MathProblem _generalProblem() {
    return MathProblem(
      question: "Qual è la capitale del Giappone?",
      answer: 'Tokyo',
    );
  }
}

class ReminderScreen extends StatefulWidget {
  const ReminderScreen({super.key});

  @override
  _ReminderScreenState createState() => _ReminderScreenState();
}

class _ReminderScreenState extends State<ReminderScreen> {
  List<Reminder> _reminders = [];
  final NotificationService _notificationService = NotificationService();
  final _intervalOptions = [10, 15, 30, 60];

  @override
  void initState() {
    super.initState();
    _loadReminders();
  }

  void _loadReminders() async {
    final prefs = await SharedPreferences.getInstance();
    List<String>? reminders = prefs.getStringList('reminders');
    
    if (reminders != null) {
      setState(() {
        _reminders = reminders.map((r) => Reminder.fromJson(r)).toList();
      });
    }
  }

  void _saveReminders() async {
    final prefs = await SharedPreferences.getInstance();
    prefs.setStringList('reminders', _reminders.map((r) => r.toJson()).toList());
  }

  void _addReminder(Reminder reminder) {
    setState(() {
      _reminders.add(reminder);
      _saveReminders();
      _scheduleReminder(reminder);
    });
  }

  void _scheduleReminder(Reminder reminder) async {
    await _notificationService.scheduleReminder(
      id: reminder.id,
      title: reminder.title,
      message: reminder.description,
      startTime: reminder.startTime,
      endTime: reminder.endTime,
      interval: reminder.interval,
    );
  }

  void _deleteReminder(int index) async {
    await _notificationService.cancelNotification(_reminders[index].id);
    setState(() {
      _reminders.removeAt(index);
      _saveReminders();
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: _reminders.isEmpty
          ? const Center(child: Text('Nessun promemoria impostato'))
          : ListView.builder(
              itemCount: _reminders.length,
              itemBuilder: (context, index) {
                final reminder = _reminders[index];
                return Card(
                  margin: const EdgeInsets.symmetric(horizontal: 10, vertical: 5),
                  child: ListTile(
                    title: Text(
                      reminder.title,
                      style: const TextStyle(fontWeight: FontWeight.bold),
                    ),
                    subtitle: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        const SizedBox(height: 5),
                        Text(reminder.description),
                        const SizedBox(height: 8),
                        Row(
                          children: [
                            const Icon(Icons.access_time, size: 16),
                            const SizedBox(width: 5),
                            Text(
                              'Inizio: ${DateFormat('HH:mm').format(reminder.startTime)}',
                            ),
                          ],
                        ),
                        if (reminder.endTime != null) ...[
                          const SizedBox(height: 4),
                          Row(
                            children: [
                              const Icon(Icons.timer_off, size: 16),
                              const SizedBox(width: 5),
                              Text(
                                'Fine: ${DateFormat('HH:mm').format(reminder.endTime!)}',
                              ),
                            ],
                          ),
                        ],
                        const SizedBox(height: 4),
                        Row(
                          children: [
                            const Icon(Icons.notifications_active, size: 16),
                            const SizedBox(width: 5),
                            Text('Notifica ogni: ${reminder.interval} min'),
                          ],
                        ),
                      ],
                    ),
                    trailing: IconButton(
                      icon: const Icon(Icons.delete, color: Colors.red),
                      onPressed: () => _deleteReminder(index),
                    ),
                  ),
                );
              },
            ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showAddReminderDialog(context),
        child: const Icon(Icons.add),
      ),
    );
  }

  void _showAddReminderDialog(BuildContext context) {
    String title = '';
    String description = '';
    DateTime startTime = DateTime.now();
    DateTime? endTime;
    int interval = _intervalOptions.first;

    showDialog(
      context: context,
      builder: (context) {
        return StatefulBuilder(
          builder: (context, setStateDialog) {
            return AlertDialog(
              title: const Text('Nuovo Promemoria'),
              content: SingleChildScrollView(
                child: Column(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    TextField(
                      decoration: const InputDecoration(
                        labelText: 'Titolo',
                        border: OutlineInputBorder(),
                      ),
                      style: const TextStyle(fontSize: 16),
                      onChanged: (value) => title = value,
                    ),
                    const SizedBox(height: 10),
                    TextField(
                      decoration: const InputDecoration(
                        labelText: 'Descrizione',
                        border: OutlineInputBorder(),
                      ),
                      style: const TextStyle(fontSize: 16),
                      onChanged: (value) => description = value,
                    ),
                    const SizedBox(height: 15),
                    ListTile(
                      contentPadding: EdgeInsets.zero,
                      leading: const Icon(Icons.access_time),
                      title: Text(
                        'Inizio: ${DateFormat('HH:mm').format(startTime)}',
                      ),
                      onTap: () async {
                        final time = await showTimePicker(
                          context: context,
                          initialTime: TimeOfDay.fromDateTime(startTime),
                        );
                        if (time != null) {
                          setStateDialog(() {
                            startTime = DateTime(
                              DateTime.now().year,
                              DateTime.now().month,
                              DateTime.now().day,
                              time.hour,
                              time.minute,
                            );
                          });
                        }
                      },
                    ),
                    const SizedBox(height: 5),
                    ListTile(
                      contentPadding: EdgeInsets.zero,
                      leading: const Icon(Icons.timer_off),
                      title: Text(endTime == null
                          ? 'Fine: Nessuna'
                          : 'Fine: ${DateFormat('HH:mm').format(endTime!)}'),
                      onTap: () async {
                        final time = await showTimePicker(
                          context: context,
                          initialTime: TimeOfDay.fromDateTime(startTime),
                        );
                        if (time != null) {
                          setStateDialog(() {
                            endTime = DateTime(
                              DateTime.now().year,
                              DateTime.now().month,
                              DateTime.now().day,
                              time.hour,
                              time.minute,
                            );
                          });
                        }
                      },
                      trailing: endTime == null 
                        ? null
                        : IconButton(
                            icon: const Icon(Icons.cancel),
                            onPressed: () {
                              setStateDialog(() => endTime = null);
                            },
                          ),
                    ),
                    const SizedBox(height: 10),
                    DropdownButtonFormField<int>(
                      value: interval,
                      items: _intervalOptions
                          .map((mins) => DropdownMenuItem(
                                value: mins,
                                child: Text('$mins minuti'),
                              ))
                          .toList(),
                      onChanged: (value) {
                        if (value != null) setStateDialog(() => interval = value);
                      },
                      decoration: const InputDecoration(
                        labelText: 'Intervallo notifiche',
                        border: OutlineInputBorder(),
                      ),
                    ),
                  ],
                ),
              ),
              actions: [
                TextButton(
                  onPressed: () => Navigator.pop(context),
                  child: const Text('Annulla'),
                ),
                TextButton(
                  onPressed: () {
                    if (title.isNotEmpty) {
                      _addReminder(Reminder(
                        id: DateTime.now().millisecondsSinceEpoch,
                        title: title,
                        description: description,
                        startTime: startTime,
                        endTime: endTime,
                        interval: interval,
                      ));
                      Navigator.pop(context);
                    }
                  },
                  child: const Text('Salva'),
                ),
              ],
            );
          },
        );
      },
    );
  }
}

class Reminder {
  final int id;
  final String title;
  final String description;
  final DateTime startTime;
  final DateTime? endTime;
  final int interval;

  Reminder({
    required this.id,
    required this.title,
    required this.description,
    required this.startTime,
    this.endTime,
    required this.interval,
  });

  String toJson() => jsonEncode({
    'id': id,
    'title': title,
    'description': description,
    'startTime': startTime.toIso8601String(),
    'endTime': endTime?.toIso8601String(),
    'interval': interval,
  });

  factory Reminder.fromJson(String jsonStr) {
    final map = jsonDecode(jsonStr);
    return Reminder(
      id: map['id'],
      title: map['title'],
      description: map['description'],
      startTime: DateTime.parse(map['startTime']),
      endTime: map['endTime'] != null && map['endTime'] != "" ? DateTime.parse(map['endTime']) : null,
      interval: map['interval'],
    );
  }
}