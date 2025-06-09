import 'dart:async';
import 'package:flutter/material.dart';
import 'package:percent_indicator/circular_percent_indicator.dart';

class Task {
  String name;
  int totalSeconds;
  int remainingSeconds;
  Timer? timer;
  bool isRunning;
  bool isExpanded;

  Task({required this.name, required this.totalSeconds})
      : remainingSeconds = totalSeconds,
        isRunning = false,
        isExpanded = false;

  void start(Function onTick) {
    isRunning = true;
    timer = Timer.periodic(Duration(seconds: 1), (t) {
      if (remainingSeconds > 0) {
        remainingSeconds--;
        onTick();
      } else {
        stop();
      }
    });
  }

  void stop() {
    timer?.cancel();
    isRunning = false;
  }

  void reset() {
    stop();
    remainingSeconds = totalSeconds;
  }

  void updateDuration(int newSeconds) {
    stop();
    totalSeconds = newSeconds;
    remainingSeconds = newSeconds;
  }

  double get progress => 1 - remainingSeconds / totalSeconds;
}

class TimerPage extends StatefulWidget {
  @override
  _TimerPageState createState() => _TimerPageState();
}

class _TimerPageState extends State<TimerPage> {
  List<Task> tasks = [];
  int selectedHour = 0;
  int selectedMinute = 0;
  int selectedSecond = 10;
  int? completedTaskIndex;

  void _addTask() {
    final nameController = TextEditingController();

    showDialog(
      context: context,
      builder: (_) {
        int tempHour = selectedHour;
        int tempMinute = selectedMinute;
        int tempSecond = selectedSecond;

        return StatefulBuilder(
          builder: (context, setModalState) {
            return AlertDialog(
              title: Text("Add Task"),
              content: SizedBox(
                width: 300,
                child: Column(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    TextField(
                      controller: nameController,
                      decoration: InputDecoration(labelText: "Task Name"),
                    ),
                    Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      children: [
                        _buildDropdown("Hour", tempHour, 0, 23, (val) {
                          setModalState(() => tempHour = val);
                        }),
                        _buildDropdown("Minute", tempMinute, 0, 59, (val) {
                          setModalState(() => tempMinute = val);
                        }),
                        _buildDropdown("Second", tempSecond, 0, 59, (val) {
                          setModalState(() => tempSecond = val);
                        }),
                      ],
                    ),
                  ],
                ),
              ),
              actions: [
                TextButton(
                  onPressed: () {
                    String name = nameController.text.trim();
                    int totalSeconds = tempHour * 3600 + tempMinute * 60 + tempSecond;

                    if (name.isEmpty) {
                      _showError("Task name cannot be empty");
                      return;
                    }
                    if (totalSeconds <= 0) {
                      _showError("Duration must be greater than 0");
                      return;
                    }

                    setState(() {
                      selectedHour = tempHour;
                      selectedMinute = tempMinute;
                      selectedSecond = tempSecond;
                      tasks.add(Task(name: name, totalSeconds: totalSeconds)..isExpanded = true);
                    });

                    Navigator.pop(context);
                  },
                  child: Text("Add"),
                )
              ],
            );
          },
        );
      },
    );
  }

  void _editTaskTime(Task task) {
    int tempHour = task.remainingSeconds ~/ 3600;
    int tempMinute = (task.remainingSeconds % 3600) ~/ 60;
    int tempSecond = task.remainingSeconds % 60;

    showDialog(
      context: context,
      builder: (_) {
        return StatefulBuilder(
          builder: (context, setModalState) {
            return AlertDialog(
              title: Text("Edit Task Time"),
              content: SizedBox(
                width: 300,
                child: Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    _buildDropdown("Hour", tempHour, 0, 23, (val) {
                      setModalState(() => tempHour = val);
                    }),
                    _buildDropdown("Minute", tempMinute, 0, 59, (val) {
                      setModalState(() => tempMinute = val);
                    }),
                    _buildDropdown("Second", tempSecond, 0, 59, (val) {
                      setModalState(() => tempSecond = val);
                    }),
                  ],
                ),
              ),
              actions: [
                TextButton(
                  onPressed: () {
                    int newSeconds = tempHour * 3600 + tempMinute * 60 + tempSecond;
                    if (newSeconds <= 0) {
                      _showError("Duration must be greater than 0");
                      return;
                    }

                    setState(() {
                      task.updateDuration(newSeconds);
                    });

                    Navigator.pop(context);
                  },
                  child: Text("Update"),
                )
              ],
            );
          },
        );
      },
    );
  }

  Widget _buildDropdown(String label, int current, int min, int max, Function(int) onChanged) {
    return Column(
      children: [
        Text(label),
        DropdownButton<int>(
          value: current,
          items: List.generate(max - min + 1, (index) {
            int val = min + index;
            return DropdownMenuItem(
              value: val,
              child: Text(val.toString().padLeft(2, '0')),
            );
          }),
          onChanged: (val) {
            if (val != null) onChanged(val);
          },
        ),
      ],
    );
  }

  void _showError(String message) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text("Error"),
        content: Text(message),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: Text("OK"),
          ),
        ],
      ),
    );
  }

  String _formatTime(int seconds) {
    int hours = seconds ~/ 3600;
    int minutes = (seconds % 3600) ~/ 60;
    int secs = seconds % 60;
    return '${hours.toString().padLeft(2, '0')}:${minutes.toString().padLeft(2, '0')}:${secs.toString().padLeft(2, '0')}';
  }

  void _onTaskComplete(int index) async {
    setState(() {
      completedTaskIndex = index;
    });

    bool dismissed = false;

    Future.delayed(Duration(seconds: 3), () {
      if (!dismissed && Navigator.canPop(context)) {
        Navigator.pop(context);
      }
    });

    await Future.delayed(Duration(milliseconds: 200));
    await showDialog(
      context: context,
      barrierDismissible: false,
      builder: (_) => AlertDialog(
        title: Text("Time's up!"),
        content: Text("Task \"${tasks[index].name}\" has completed."),
        actions: [
          TextButton(
            onPressed: () {
              dismissed = true;
              Navigator.pop(context);
            },
            child: Text("OK"),
          )
        ],
      ),
    );

    setState(() {
      completedTaskIndex = null;
    });
  }

  @override
  void dispose() {
    for (var task in tasks) {
      task.stop();
    }
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Color(0xFF1F1F2E),
      appBar: AppBar(
        backgroundColor: Colors.black,
        title: Text('Multi-task Timer'),
        actions: [
          IconButton(
            icon: Icon(Icons.add),
            onPressed: _addTask,
          )
        ],
      ),
      body: tasks.isEmpty
          ? Center(child: Text("No tasks yet", style: TextStyle(color: Colors.white70)))
          : ListView.builder(
              itemCount: tasks.length,
              itemBuilder: (_, index) {
                final task = tasks[index];
                final isComplete = completedTaskIndex == index;
                return Dismissible(
                  key: Key(task.name + index.toString()),
                  direction: DismissDirection.endToStart,
                  background: Container(
                    alignment: Alignment.centerRight,
                    padding: EdgeInsets.symmetric(horizontal: 20),
                    color: Colors.redAccent,
                    child: Icon(Icons.delete, color: Colors.white),
                  ),
                  onDismissed: (_) {
                    setState(() {
                      task.stop();
                      tasks.removeAt(index);
                    });
                  },
                  child: ExpansionTile(
                    tilePadding: EdgeInsets.symmetric(horizontal: 16),
                    collapsedIconColor: Colors.white70,
                    iconColor: Colors.white,
                    backgroundColor: Color(0xFF2B2B3D),
                    collapsedBackgroundColor: Color(0xFF2B2B3D),
                    title: Text(task.name,
                        style: TextStyle(fontSize: 18, color: Colors.white)),
                    initiallyExpanded: task.isExpanded,
                    onExpansionChanged: (expanded) {
                      setState(() {
                        task.isExpanded = expanded;
                      });
                    },
                    children: [
                      Padding(
                        padding: const EdgeInsets.only(bottom: 20),
                        child: TweenAnimationBuilder<double>(
                          duration: Duration(milliseconds: 300),
                          tween: Tween(begin: 1.0, end: isComplete ? 1.6 : 1.0),
                          curve: Curves.elasticOut,
                          builder: (context, scale, child) {
                            return Transform.scale(
                              scale: scale,
                              child: Opacity(
                                opacity: isComplete ? 0.6 : 1.0,
                                child: Container(
                                  decoration: isComplete
                                      ? BoxDecoration(
                                          border: Border.all(color: Colors.amber, width: 4),
                                          borderRadius: BorderRadius.circular(150),
                                        )
                                      : null,
                                  child: CircularPercentIndicator(
                                    radius: 120,
                                    lineWidth: 16,
                                    percent: task.progress.clamp(0.0, 1.0),
                                    animation: true,
                                    animateFromLastPercent: true,
                                    circularStrokeCap: CircularStrokeCap.round,
                                    backgroundColor: Colors.grey.shade800,
                                    linearGradient: LinearGradient(
                                      colors: [Colors.purpleAccent, Colors.lightBlueAccent],
                                    ),
                                    center: Column(
                                      mainAxisAlignment: MainAxisAlignment.center,
                                      children: [
                                        Icon(Icons.timer, size: 30, color: Colors.white),
                                        Text(
                                          _formatTime(task.remainingSeconds),
                                          style: TextStyle(
                                            fontSize: 24,
                                            fontWeight: FontWeight.bold,
                                            color: Colors.white,
                                          ),
                                        ),
                                        Text("Time Remaining",
                                            style: TextStyle(color: Colors.white70)),
                                      ],
                                    ),
                                  ),
                                ),
                              ),
                            );
                          },
                        ),
                      ),
                      Row(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          ElevatedButton.icon(
                            onPressed: () {
                              setState(() {
                                if (task.isRunning) {
                                  task.stop();
                                } else {
                                  task.start(() {
                                    setState(() {});
                                    if (task.remainingSeconds == 0) {
                                      _onTaskComplete(index);
                                    }
                                  });
                                }
                              });
                            },
                            icon: Icon(task.isRunning ? Icons.pause : Icons.play_arrow),
                            label: Text(task.isRunning ? "Pause" : "Start"),
                          ),
                          SizedBox(width: 10),
                          ElevatedButton.icon(
                            onPressed: () {
                              setState(() {
                                task.reset();
                              });
                            },
                            icon: Icon(Icons.restart_alt),
                            label: Text("Reset"),
                            style: ElevatedButton.styleFrom(
                              backgroundColor: Colors.redAccent,
                            ),
                          ),
                          SizedBox(width: 10),
                          ElevatedButton.icon(
                            onPressed: () {
                              _editTaskTime(task);
                            },
                            icon: Icon(Icons.edit),
                            label: Text("Edit"),
                            style: ElevatedButton.styleFrom(
                              backgroundColor: Colors.orange,
                            ),
                          ),
                        ],
                      ),
                    ],
                  ),
                );
              },
            ),
    );
  }
}
