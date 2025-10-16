#!/bin/bash

TASK_FILE="tasks.txt"
touch "$TASK_FILE"

# Preload some tasks (only if file is empty)
if [ ! -s "$TASK_FILE" ]; then
    echo "Alice | alice@example.com | PENDING | Finish Linux lab | 2025-10-16 15:30" >> "$TASK_FILE"
    echo "Bob   | bob@example.com   | PENDING | Study AWK       | 2025-10-16 16:00" >> "$TASK_FILE"
    echo "Charlie | charlie@example.com | PENDING | Prepare report | 2025-10-17 10:00" >> "$TASK_FILE"

    # Schedule reminders automatically
    awk -F"|" '{
        user=$1; email=$2; task=$4; datetime=$5
        split(datetime, a, " ")
        split(a[1], d, "-")
        split(a[2], t, ":")
        task_time=t[1]":"t[2]" "d[2]"/"d[3]"/"d[1]
        system("echo notify-send \"Reminder for "user"\" \""task"\" | at \""task_time"\" 2>/dev/null")
        system("echo \"Reminder: "task"\" | mail -s \"Task Reminder for "user"\" "email")")
    }' "$TASK_FILE"
fi

echo "=== Multi-User To-Do & Reminder Manager ==="
read -p "Enter your name: " user
read -p "Enter your email: " email

echo "1) Add Task"
echo "2) List My Tasks"
echo "3) Schedule Reminder for My Tasks"
read -p "Choose option [1-3]: " choice

case $choice in
  1)
    read -p "Enter task description: " task
    read -p "Enter reminder date and time (YYYY-MM-DD HH:MM): " datetime
    echo "$user | $email | PENDING | $task | $datetime" >> "$TASK_FILE"
    echo "Task added for $user with reminder at $datetime!"

    # Convert to at-compatible format
    split_date=$(echo $datetime | awk '{print $1}')
    split_time=$(echo $datetime | awk '{print $2}')
    IFS="-" read -r y m d <<< "$split_date"
    IFS=":" read -r h min <<< "$split_time"
    task_time="$h:$min $m/$d/$y"

    # Schedule desktop notification
    at_output=$(echo "notify-send 'Reminder for $user' '$task'" | at "$task_time" 2>&1)
    # Schedule email notification
    at_output=$(echo "echo 'Reminder: $task' | mail -s 'Task Reminder for $user' $email" | at "$task_time" 2>&1)

    echo "Reminder for task '$task' scheduled at $datetime (desktop + email)"
    ;;

  2)
    echo "=== Tasks for $user ==="
    awk -F"|" -v u="$user" '{if($1==u) printf "%s. [%s] %s (Reminder: %s)\n", NR, $3, $4, $5}' "$TASK_FILE"
    ;;

  3)
    echo "=== Scheduling Pending Reminders for $user ==="
    awk -F"|" -v u="$user" -v e="$email" '$1==u && $3=="PENDING" {print $4 "|" $5}' "$TASK_FILE" | while IFS="|" read task datetime
    do
        split_date=$(echo $datetime | awk '{print $1}')
        split_time=$(echo $datetime | awk '{print $2}')
        IFS="-" read -r y m d <<< "$split_date"
        IFS=":" read -r h min <<< "$split_time"
        task_time="$h:$min $m/$d/$y"

        # Desktop notification
        at_output=$(echo "notify-send 'Reminder for $u' '$task'" | at "$task_time" 2>&1)
        # Email notification
        at_output=$(echo "echo 'Reminder: $task' | mail -s 'Task Reminder for $u' $email" | at "$task_time" 2>&1)

        echo "Reminder scheduled for task '$task' at $datetime (desktop + email)"
    done
    ;;

  *)
    echo "Invalid option"
    ;;
esac

