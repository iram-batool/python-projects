# python-projects
A Python CLI app to manage student records (add, update, delete, list) with JSON persistence, OOP design, and input validation.
## What I Learned

- Implementing OOP principles (encapsulation, class methods, static methods)
- Working with file I/O and JSON serialization in Python
- Writing validation logic using regular expressions
- Designing a clean, user-friendly CLI interface

## Author

Built by [Iram batool] as part of a Python learning roadmap toward AI/web development.

"""
Student Management System
CLI app with OOP, file persistence (JSON), and validation.
"""

import json
import os
import re

DATA_FILE = "students.json"

# ─────────────────────────────────────────
#  Student class
# ─────────────────────────────────────────
class Student:
    def __init__(self, student_id: str, name: str, grade: float):
        self.student_id = student_id.strip().upper()
        self.name = name.strip().title()
        self.grade = float(grade)

    def to_dict(self) -> dict:
        return {
            "student_id": self.student_id,
            "name": self.name,
            "grade": self.grade
        }

    @classmethod
    def from_dict(cls, data: dict) -> "Student":
        return cls(data["student_id"], data["name"], data["grade"])

    def grade_letter(self) -> str:
        g = self.grade
        if g >= 90: return "A+"
        if g >= 80: return "A"
        if g >= 70: return "B"
        if g >= 60: return "C"
        if g >= 50: return "D"
        return "F"

    def __str__(self) -> str:
        return (f"  ID    : {self.student_id}\n"
                f"  Name  : {self.name}\n"
                f"  Grade : {self.grade:.1f}  ({self.grade_letter()})")


# ─────────────────────────────────────────
#  StudentManager class
# ─────────────────────────────────────────
class StudentManager:
    def __init__(self, filepath: str = DATA_FILE):
        self.filepath = filepath
        self.students: dict[str, Student] = {}
        self._load()

    # ── Persistence ──────────────────────
    def _load(self):
        if os.path.exists(self.filepath):
            try:
                with open(self.filepath, "r", encoding="utf-8") as f:
                    raw = json.load(f)
                self.students = {
                    item["student_id"]: Student.from_dict(item)
                    for item in raw
                }
            except (json.JSONDecodeError, KeyError):
                self.students = {}

    def _save(self):
        with open(self.filepath, "w", encoding="utf-8") as f:
            json.dump(
                [s.to_dict() for s in self.students.values()],
                f, indent=2, ensure_ascii=False
            )

    # ── Validation ───────────────────────
    @staticmethod
    def _validate_id(sid: str) -> bool:
        """ID must be non-empty alphanumeric, 2-10 chars."""
        return bool(re.fullmatch(r"[A-Za-z0-9]{2,10}", sid.strip()))

    @staticmethod
    def _validate_grade(grade_str: str) -> tuple[bool, float]:
        try:
            g = float(grade_str)
            return (0 <= g <= 100), g
        except ValueError:
            return False, 0.0

    # ── CRUD operations ──────────────────
    def add(self, student_id: str, name: str, grade_str: str) -> str:
        if not self._validate_id(student_id):
            return "❌  Invalid ID. Use 2–10 alphanumeric characters."
        sid = student_id.strip().upper()
        if sid in self.students:
            return f"❌  ID '{sid}' already exists. Use 'update' to change it."
        if not name.strip():
            return "❌  Name cannot be empty."
        ok, grade = self._validate_grade(grade_str)
        if not ok:
            return "❌  Grade must be a number between 0 and 100."
        self.students[sid] = Student(sid, name, grade)
        self._save()
        return f"✅  Student '{sid}' added successfully."

    def update(self, student_id: str,
               new_name: str = "", new_grade_str: str = "") -> str:
        sid = student_id.strip().upper()
        if sid not in self.students:
            return f"❌  No student with ID '{sid}' found."
        s = self.students[sid]
        if new_name.strip():
            s.name = new_name.strip().title()
        if new_grade_str.strip():
            ok, grade = self._validate_grade(new_grade_str)
            if not ok:
                return "❌  Grade must be a number between 0 and 100."
            s.grade = grade
        self._save()
        return f"✅  Student '{sid}' updated."

    def delete(self, student_id: str) -> str:
        sid = student_id.strip().upper()
        if sid not in self.students:
            return f"❌  No student with ID '{sid}' found."
        del self.students[sid]
        self._save()
        return f"✅  Student '{sid}' deleted."

    def get(self, student_id: str) -> str:
        sid = student_id.strip().upper()
        if sid not in self.students:
            return f"❌  No student with ID '{sid}' found."
        return f"\n{'─'*35}\n{self.students[sid]}\n{'─'*35}"

    def list_all(self) -> str:
        if not self.students:
            return "ℹ️   No students on record yet."
        lines = [f"\n{'─'*50}",
                 f"  {'ID':<12} {'Name':<20} {'Grade':>7}  {'Letter':>6}",
                 f"{'─'*50}"]
        for s in sorted(self.students.values(), key=lambda x: x.student_id):
            lines.append(
                f"  {s.student_id:<12} {s.name:<20} {s.grade:>6.1f}%  {s.grade_letter():>6}"
            )
        lines.append(f"{'─'*50}")
        lines.append(f"  Total: {len(self.students)} student(s)")
        return "\n".join(lines)


# ─────────────────────────────────────────
#  CLI helpers
# ─────────────────────────────────────────
def prompt(text: str) -> str:
    return input(f"  {text}: ").strip()

def banner():
    print("\n" + "═"*50)
    print("       🎓  STUDENT MANAGEMENT SYSTEM")
    print("═"*50)

def menu():
    print("""
  [1]  Add student
  [2]  Update student
  [3]  Delete student
  [4]  Search by ID
  [5]  List all students
  [0]  Exit
""")

# ─────────────────────────────────────────
#  Main loop
# ─────────────────────────────────────────
def main():
    manager = StudentManager()
    banner()
    print(f"  Data file: {os.path.abspath(DATA_FILE)}")

    while True:
        menu()
        choice = input("  Enter choice: ").strip()

        if choice == "1":
            print("\n  ── Add Student ──")
            sid   = prompt("Student ID (e.g. S001)")
            name  = prompt("Full name")
            grade = prompt("Grade (0–100)")
            print(f"\n  {manager.add(sid, name, grade)}")

        elif choice == "2":
            print("\n  ── Update Student ──")
            sid   = prompt("Student ID to update")
            name  = prompt("New name  (press Enter to skip)")
            grade = prompt("New grade (press Enter to skip)")
            print(f"\n  {manager.update(sid, name, grade)}")

        elif choice == "3":
            print("\n  ── Delete Student ──")
            sid = prompt("Student ID to delete")
            confirm = input(f"  Confirm delete '{sid.upper()}'? (y/n): ")
            if confirm.lower() == "y":
                print(f"\n  {manager.delete(sid)}")
            else:
                print("  ↩️  Cancelled.")

        elif choice == "4":
            print("\n  ── Search Student ──")
            sid = prompt("Student ID")
            print(manager.get(sid))

        elif choice == "5":
            print(manager.list_all())

        elif choice == "0":
            print("\n  Goodbye! 👋\n")
            break

        else:
            print("  ⚠️  Invalid choice. Enter 0–5.")


if __name__ == "__main__":
    main()
