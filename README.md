# Student-Management-System---project---StaxTech

rt sqlite3
from datetime import datetime
from typing import Optional, List, Tuple

DB_FILE = "sms.db"


def get_conn():
    return sqlite3.connect(DB_FILE)

def init_db():
    conn = get_conn()
    cur = conn.cursor()

    
    cur.execute("""
    CREATE TABLE IF NOT EXISTS students (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        reg_no TEXT UNIQUE NOT NULL,
        name TEXT NOT NULL,
        email TEXT,
        phone TEXT,
        year INTEGER,
        department TEXT
    );
    """)

   
    cur.execute("""
    CREATE TABLE IF NOT EXISTS courses (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        code TEXT UNIQUE NOT NULL,
        title TEXT NOT NULL,
        credits INTEGER NOT NULL,
        capacity INTEGER NOT NULL
    );
    """)

    cur.execute("""
    CREATE TABLE IF NOT EXISTS enrollments (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        student_id INTEGER NOT NULL,
        course_id INTEGER NOT NULL,
        enrolled_on TEXT NOT NULL,
        UNIQUE(student_id, course_id),
        FOREIGN KEY(student_id) REFERENCES students(id) ON DELETE CASCADE,
        FOREIGN KEY(course_id) REFERENCES courses(id) ON DELETE CASCADE
    );
    """)

    
    cur.execute("""
    CREATE TABLE IF NOT EXISTS attendance (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        student_id INTEGER NOT NULL,
        course_id INTEGER NOT NULL,
        date TEXT NOT NULL,
        status TEXT NOT NULL CHECK (status IN ('Present','Absent','Late','Excused')),
        UNIQUE(student_id, course_id, date),
        FOREIGN KEY(student_id) REFERENCES students(id) ON DELETE CASCADE,
        FOREIGN KEY(course_id) REFERENCES courses(id) ON DELETE CASCADE
    );
    """)

    
    cur.execute("""
    CREATE TABLE IF NOT EXISTS grades (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        student_id INTEGER NOT NULL,
        course_id INTEGER NOT NULL,
        component TEXT NOT NULL, -- e.g., 'Midterm', 'Final', 'Assignment 1'
        score REAL NOT NULL,     -- 0-100 recommended
        max_score REAL NOT NULL, -- default 100
        recorded_on TEXT NOT NULL,
        FOREIGN KEY(student_id) REFERENCES students(id) ON DELETE CASCADE,
        FOREIGN KEY(course_id) REFERENCES courses(id) ON DELETE CASCADE
    );
    """)

    
    cur.execute("""
    CREATE TABLE IF NOT EXISTS communications (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        student_id INTEGER NOT NULL,
        contact_type TEXT NOT NULL CHECK (contact_type IN ('Parent','Teacher','Student')),
        medium TEXT NOT NULL CHECK (medium IN ('Email','Phone','In-Person','Message')),
        subject TEXT,
        notes TEXT,
        timestamp TEXT NOT NULL,
        FOREIGN KEY(student_id) REFERENCES students(id) ON DELETE CASCADE
    );
    """)

    conn.commit()
    conn.close()


def now_str() -> str:
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

def input_int(prompt: str, allow_empty=False) -> Optional[int]:
    while True:
        val = input(prompt).strip()
        if allow_empty and val == "":
            return None
        try:
            return int(val)
        except ValueError:
            print("Please enter a valid integer.")

def input_nonempty(prompt: str) -> str:
    while True:
        val = input(prompt).strip()
        if val:
            return val
        print("This field cannot be empty.")


def add_student():
    reg_no = input_nonempty("Registration No: ")
    name = input_nonempty("Name: ")
    email = input("Email (optional): ").strip() or None
    phone = input("Phone (optional): ").strip() or None
    year = input_int("Year (e.g., 1-4, optional): ", allow_empty=True)
    department = input("Department (optional): ").strip() or None

    conn = get_conn()
    cur = conn.cursor()
    try:
        cur.execute("""
            INSERT INTO students (reg_no, name, email, phone, year, department)
            VALUES (?, ?, ?, ?, ?, ?)
        """, (reg_no, name, email, phone, year, department))
        conn.commit()
        print(f"Student '{name}' added.")
    except sqlite3.IntegrityError as e:
        print(f"Error: {e}. Registration No must be unique.")
    finally:
        conn.close()

def list_students():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id, reg_no, name, email, phone, year, department FROM students ORDER BY id")
    rows = cur.fetchall()
    conn.close()
    if not rows:
        print("No students found.")
        return
    print("\nStudents:")
    for r in rows:
        print(f"[{r[0]}] {r[2]} (Reg: {r[1]}, Year: {r[5]}, Dept: {r[6]}, Email: {r[3]}, Phone: {r[4]})")

def update_student():
    list_students()
    sid = input_int("Enter Student ID to update: ")
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id, reg_no, name, email, phone, year, department FROM students WHERE id=?", (sid,))
    row = cur.fetchone()
    if not row:
        print("Student not found.")
        conn.close()
        return

    print("Leave blank to keep existing.")
    reg_no = input(f"Registration No [{row[1]}]: ").strip() or row[1]
    name = input(f"Name [{row[2]}]: ").strip() or row[2]
    email = input(f"Email [{row[3] or ''}]: ").strip() or row[3]
    phone = input(f"Phone [{row[4] or ''}]: ").strip() or row[4]
    year_str = input(f"Year [{row[5] if row[5] is not None else ''}]: ").strip()
    year = int(year_str) if year_str else row[5]
    department = input(f"Department [{row[6] or ''}]: ").strip() or row[6]

    try:
        cur.execute("""
            UPDATE students SET reg_no=?, name=?, email=?, phone=?, year=?, department=? WHERE id=?
        """, (reg_no, name, email, phone, year, department, sid))
        conn.commit()
        print("Student updated.")
    except sqlite3.IntegrityError as e:
        print(f"Error: {e}. Registration No must be unique.")
    finally:
        conn.close()

def delete_student():
    list_students()
    sid = input_int("Enter Student ID to delete: ")
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("DELETE FROM students WHERE id=?", (sid,))
    conn.commit()
    conn.close()
    print("Student deleted (if existed).")


def add_course():
    code = input_nonempty("Course Code (unique): ")
    title = input_nonempty("Course Title: ")
    credits = input_int("Credits (e.g., 3): ")
    capacity = input_int("Capacity: ")
    conn = get_conn()
    cur = conn.cursor()
    try:
        cur.execute("""
            INSERT INTO courses (code, title, credits, capacity)
            VALUES (?, ?, ?, ?)
        """, (code, title, credits, capacity))
        conn.commit()
        print(f"Course '{title}' added.")
    except sqlite3.IntegrityError as e:
        print(f"Error: {e}. Course Code must be unique.")
    finally:
        conn.close()

def list_courses():
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id, code, title, credits, capacity FROM courses ORDER BY id")
    rows = cur.fetchall()
    conn.close()
    if not rows:
        print("No courses found.")
        return
    print("\nCourses:")
    for r in rows:
        print(f"[{r[0]}] {r[2]} (Code: {r[1]}, Credits: {r[3]}, Capacity: {r[4]})")

def update_course():
    list_courses()
    cid = input_int("Enter Course ID to update: ")
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id, code, title, credits, capacity FROM courses WHERE id=?", (cid,))
    row = cur.fetchone()
    if not row:
        print("Course not found.")
        conn.close()
        return

    print("Leave blank to keep existing.")
    code = input(f"Code [{row[1]}]: ").strip() or row[1]
    title = input(f"Title [{row[2]}]: ").strip() or row[2]
    credits_str = input(f"Credits [{row[3]}]: ").strip()
    credits = int(credits_str) if credits_str else row[3]
    capacity_str = input(f"Capacity [{row[4]}]: ").strip()
    capacity = int(capacity_str) if capacity_str else row[4]

    try:
        cur.execute("""
            UPDATE courses SET code=?, title=?, credits=?, capacity=? WHERE id=?
        """, (code, title, credits, capacity, cid))
        conn.commit()
        print("Course updated.")
    except sqlite3.IntegrityError as e:
        print(f"Error: {e}. Course Code must be unique.")
    finally:
        conn.close()

def delete_course():
    list_courses()
    cid = input_int("Enter Course ID to delete: ")
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("DELETE FROM courses WHERE id=?", (cid,))
    conn.commit()
    conn.close()
    print("Course deleted (if existed).")


def enroll_student():
    list_students()
    sid = input_int("Student ID to enroll: ")
    list_courses()
    cid = input_int("Course ID to enroll into: ")

    conn = get_conn()
    cur = conn.cursor()

   
    cur.execute("SELECT capacity FROM courses WHERE id=?", (cid,))
    course = cur.fetchone()
    if not course:
        print("Course not found.")
        conn.close()
        return

    capacity = course[0]
    cur.execute("SELECT COUNT(*) FROM enrollments WHERE course_id=?", (cid,))
    count = cur.fetchone()[0]
    if count >= capacity:
        print("Cannot enroll: capacity full.")
        conn.close()
        return

    try:
        cur.execute("""
            INSERT INTO enrollments (student_id, course_id, enrolled_on) VALUES (?, ?, ?)
        """, (sid, cid, now_str()))
        conn.commit()
        print("Enrollment successful.")
    except sqlite3.IntegrityError:
        print("Student is already enrolled in this course.")
    finally:
        conn.close()

def unenroll_student():
    list_students()
    sid = input_int("Student ID to unenroll: ")
    list_courses()
    cid = input_int("Course ID to remove: ")

    conn = get_conn()
    cur = conn.cursor()
    cur.execute("DELETE FROM enrollments WHERE student_id=? AND course_id=?", (sid, cid))
    conn.commit()
    conn.close()
    print("Unenrollment completed (if existed).")

def list_enrollments_for_student():
    list_students()
    sid = input_int("Student ID: ")
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        SELECT c.id, c.code, c.title
        FROM enrollments e
        JOIN courses c ON e.course_id = c.id
        WHERE e.student_id=?
        ORDER BY c.code
    """, (sid,))
    rows = cur.fetchall()
    conn.close()
    if not rows:
        print("No enrollments found for this student.")
        return
    print("\nEnrollments:")
    for r in rows:
        print(f"[{r[0]}] {r[2]} (Code: {r[1]})")

def list_students_in_course():
    list_courses()
    cid = input_int("Course ID: ")
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        SELECT s.id, s.reg_no, s.name
        FROM enrollments e
        JOIN students s ON e.student_id = s.id
        WHERE e.course_id=?
        ORDER BY s.name
    """, (cid,))
    rows = cur.fetchall()
    conn.close()
    if not rows:
        print("No students enrolled.")
        return
    print("\nStudents in course:")
    for r in rows:
        print(f"[{r[0]}] {r[2]} (Reg: {r[1]})")


def mark_attendance():
    list_courses()
    cid = input_int("Course ID: ")
    date = input("Date (YYYY-MM-DD, default today): ").strip()
    if not date:
        date = datetime.now().strftime("%Y-%m-%d")

    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        SELECT s.id, s.name
        FROM enrollments e
        JOIN students s ON s.id = e.student_id
        WHERE e.course_id=?
        ORDER BY s.name
    """, (cid,))
    students = cur.fetchall()
    if not students:
        print("No students enrolled in this course.")
        conn.close()
        return

    print(f"\nMarking attendance for course {cid} on {date}:")
    print("Enter P for Present, A for Absent, L for Late, E for Excused")
    status_map = {'P': 'Present', 'A': 'Absent', 'L': 'Late', 'E': 'Excused'}

    for sid, name in students:
        while True:
            val = input(f"{name}: ").strip().upper()
            if val in status_map:
                status = status_map[val]
                try:
                    cur.execute("""
                        INSERT INTO attendance (student_id, course_id, date, status)
                        VALUES (?, ?, ?, ?)
                    """, (sid, cid, date, status))
                    conn.commit()
                except sqlite3.IntegrityError:
                    # Update if already marked
                    cur.execute("""
                        UPDATE attendance SET status=? WHERE student_id=? AND course_id=? AND date=?
                    """, (status, sid, cid, date))
                    conn.commit()
                break
            else:
                print("Invalid. Use P/A/L/E.")
    conn.close()
    print("Attendance saved.")

def view_attendance_report():
    list_courses()
    cid = input_int("Course ID: ")
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        SELECT s.name, a.date, a.status
        FROM attendance a
        JOIN students s ON s.id=a.student_id
        WHERE a.course_id=?
        ORDER BY a.date, s.name
    """, (cid,))
    rows = cur.fetchall()
    conn.close()
    if not rows:
        print("No attendance records.")
        return
    print("\nAttendance report:")
    for name, date, status in rows:
        print(f"{date} - {name}: {status}")


def record_grade():
    list_students()
    sid = input_int("Student ID: ")
    list_courses()
    cid = input_int("Course ID: ")
    component = input_nonempty("Component (e.g., Midterm/Final/Assignment 1): ")
    score = float(input_nonempty("Score (e.g., 85): "))
    max_score_str = input("Max score (default 100): ").strip()
    max_score = float(max_score_str) if max_score_str else 100.0

    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        INSERT INTO grades (student_id, course_id, component, score, max_score, recorded_on)
        VALUES (?, ?, ?, ?, ?, ?)
    """, (sid, cid, component, score, max_score, now_str()))
    conn.commit()
    conn.close()
    print("Grade recorded.")

def view_grades_for_student():
    list_students()
    sid = input_int("Student ID: ")

    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        SELECT c.code, c.title, g.component, g.score, g.max_score, g.recorded_on
        FROM grades g
        JOIN courses c ON c.id=g.course_id
        WHERE g.student_id=?
        ORDER BY c.code, g.recorded_on
    """, (sid,))
    rows = cur.fetchall()
    conn.close()
    if not rows:
        print("No grades recorded.")
        return
    print("\nGrades:")
    for code, title, comp, score, max_s, when in rows:
        pct = (score / max_s) * 100 if max_s else 0
        print(f"{code} - {title} | {comp}: {score}/{max_s} ({pct:.1f}%) on {when}")


def add_communication():
    list_students()
    sid = input_int("Student ID: ")
    print("Contact type options: Parent, Teacher, Student")
    contact_type = input_nonempty("Contact type: ").title()
    print("Medium options: Email, Phone, In-Person, Message")
    medium = input_nonempty("Medium: ").title()
    subject = input("Subject (optional): ").strip() or None
    notes = input("Notes: ").strip() or None

    if contact_type not in ("Parent","Teacher","Student"):
        print("Invalid contact type.")
        return
    if medium not in ("Email","Phone","In-Person","Message"):
        print("Invalid medium.")
        return

    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        INSERT INTO communications (student_id, contact_type, medium, subject, notes, timestamp)
        VALUES (?, ?, ?, ?, ?, ?)
    """, (sid, contact_type, medium, subject, notes, now_str()))
    conn.commit()
    conn.close()
    print("Communication logged.")

def view_communications_for_student():
    list_students()
    sid = input_int("Student ID: ")
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("""
        SELECT contact_type, medium, subject, notes, timestamp
        FROM communications
        WHERE student_id=?
        ORDER BY timestamp DESC
    """, (sid,))
    rows = cur.fetchall()
    conn.close()
    if not rows:
        print("No communication logs.")
        return
    print("\nCommunications:")
    for ctype, medium, subject, notes, ts in rows:
        subj = subject or "-"
        note = notes or "-"
        print(f"[{ts}] {ctype} via {medium} | {subj} -> {note}")


def student_summary():
    list_students()
    sid = input_int("Student ID: ")
    conn = get_conn()
    cur = conn.cursor()

    cur.execute("SELECT reg_no, name, email, phone, year, department FROM students WHERE id=?", (sid,))
    s = cur.fetchone()
    if not s:
        print("Student not found.")
        conn.close()
        return
    reg_no, name, email, phone, year, dept = s

    cur.execute("""
        SELECT COUNT(*)
        FROM enrollments
        WHERE student_id=?
    """, (sid,))
    course_count = cur.fetchone()[0]

    cur.execute("""
        SELECT COUNT(*) FILTER (WHERE status='Present'),
               COUNT(*) FILTER (WHERE status='Absent'),
               COUNT(*)
        FROM attendance
        WHERE student_id=?
    """, (sid,))
    try:
        present, absent, total = cur.fetchone()
    except sqlite3.OperationalError:
       
        cur.execute("SELECT status FROM attendance WHERE student_id=?", (sid,))
        statuses = [row[0] for row in cur.fetchall()]
        present = sum(1 for st in statuses if st == "Present")
        absent = sum(1 for st in statuses if st == "Absent")
        total = len(statuses)

    cur.execute("""
        SELECT AVG((score/max_score)*100)
        FROM grades
        WHERE student_id=?
    """, (sid,))
    avg_pct = cur.fetchone()[0]

    conn.close()
    print(f"\nStudent Summary: {name} (Reg: {reg_no})")
    print(f"Year: {year} | Dept: {dept} | Email: {email} | Phone: {phone}")
    print(f"Enrolled courses: {course_count}")
    if total:
        print(f"Attendance: Present {present}/{total}, Absent {absent}/{total}")
    else:
        print("Attendance: No records")
    if avg_pct is not None:
        print(f"Average grade: {avg_pct:.1f}%")
    else:
        print("Average grade: No records")

def course_summary():
    list_courses()
    cid = input_int("Course ID: ")
    conn = get_conn()
    cur = conn.cursor()

    cur.execute("SELECT code, title, credits, capacity FROM courses WHERE id=?", (cid,))
    c = cur.fetchone()
    if not c:
        print("Course not found.")
        conn.close()
        return
    code, title, credits, capacity = c

    cur.execute("SELECT COUNT(*) FROM enrollments WHERE course_id=?", (cid,))
    enrolled = cur.fetchone()[0]

    cur.execute("""
        SELECT COUNT(*) FILTER (WHERE status='Present'),
               COUNT(*) FILTER (WHERE status='Absent'),
               COUNT(*)
        FROM attendance
        WHERE course_id=?
    """, (cid,))
    try:
        present, absent, total = cur.fetchone()
    except sqlite3.OperationalError:
        cur.execute("SELECT status FROM attendance WHERE course_id=?", (cid,))
        statuses = [row[0] for row in cur.fetchall()]
        present = sum(1 for st in statuses if st == "Present")
        absent = sum(1 for st in statuses if st == "Absent")
        total = len(statuses)

    conn.close()
    print(f"\nCourse Summary: {title} ({code})")
    print(f"Credits: {credits} | Capacity: {capacity} | Enrolled: {enrolled}")
    if total:
        print(f"Attendance entries: {total} | Present: {present} | Absent: {absent}")
    else:
        print("Attendance: No records")


def students_menu():
    while True:
        print("\n--- Students ---")
        print("1. Add student")
        print("2. List students")
        print("3. Update student")
        print("4. Delete student")
        print("5. Back")
        choice = input("Choose: ").strip()
        if choice == "1": add_student()
        elif choice == "2": list_students()
        elif choice == "3": update_student()
        elif choice == "4": delete_student()
        elif choice == "5": break
        else: print("Invalid choice.")
