#!/usr/bin/env python3
# =============================================================================
# Group Randomizer
# Splits participants into groups either purely randomly or balanced by score.
#
# Mode 1 - Random: pure shuffle, splits as evenly as possible
# Mode 2 - Balanced: uses snake-draft algorithm so each group gets
#           similar total score (like a fantasy sports draft)
#
# File format for save/load JSON:
#   {
#     "participants": ["Alice", "Bob", ...],
#     "scores": {"Alice": 8, "Bob": 5, ...}  <- only used in balanced mode
#   }
#
# To run: python3 group_randomizer.py
# =============================================================================

import tkinter as tk
from tkinter import ttk, messagebox, filedialog
from typing import List, Tuple
import random
import json
import csv


# ── BALANCING ALGORITHM ───────────────────────────────────────────────────────
# Snake draft: sort by score descending, then assign in a snake pattern.
# Round 1: group 0, 1, 2 ...
# Round 2: ... 2, 1, 0  (reverses)
# This mirrors how sports leagues draft players to keep teams balanced.

def balanced_split(students, group_count):
    # students = list of (name, score) tuples
    # Shuffle first so people with equal scores don't always land in the same group.
    # Then sort by score descending — shuffle preserves randomness within same-score tiers.
    shuffled_first = students[:]
    random.shuffle(shuffled_first)
    sorted_students = sorted(shuffled_first, key=lambda x: x[1], reverse=True)
    groups: List[List[Tuple[str, int]]] = [[] for _ in range(group_count)]
    for i, student in enumerate(sorted_students):
        # snake direction: even rounds go forward, odd rounds go backward
        if (i // group_count) % 2 == 0:
            idx = i % group_count
        else:
            idx = group_count - 1 - (i % group_count)
        groups[idx].append(student)
    return groups


def random_split(names, group_count):
    shuffled = names[:]
    random.shuffle(shuffled)
    # distribute one by one so groups differ by at most 1 person
    groups: List[List[str]] = [[] for _ in range(group_count)]
    for i, name in enumerate(shuffled):
        groups[i % group_count].append(name)
    return groups


# ── SCORE EDITOR POPUP ────────────────────────────────────────────────────────
# Opens a separate window where user sets each participant's score (1-10).
# Returns a dict of {name: score} when confirmed, or None if cancelled.

class ScoreEditor(tk.Toplevel):
    def __init__(self, parent, participants, existing_scores):
        super().__init__(parent)
        self.title("Set Scores")
        self.resizable(False, False)
        self.grab_set()  # make it modal (blocks main window until closed)
        self.result = None  # will hold {name: score} on confirm

        self.score_vars = {}  # {name: IntVar}

        # ── title ──
        tk.Label(self, text="Set each participant's score (1–10)",
                 font=("TkDefaultFont", 11, "bold"), pady=8).pack()

        # ── scrollable list of name + spinbox rows ──
        container = tk.Frame(self)
        container.pack(fill="both", expand=True, padx=16, pady=4)

        canvas = tk.Canvas(container, width=300, height=min(400, len(participants) * 36 + 10))
        scrollbar = ttk.Scrollbar(container, orient="vertical", command=canvas.yview)
        scroll_frame = tk.Frame(canvas)

        scroll_frame.bind("<Configure>",
            lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.create_window((0, 0), window=scroll_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)

        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        for name in participants:
            row = tk.Frame(scroll_frame)
            row.pack(fill="x", pady=2)

            tk.Label(row, text=name, width=20, anchor="w").pack(side="left")

            var = tk.IntVar(value=existing_scores.get(name, 5))  # default score = 5
            self.score_vars[name] = var

            # Spinbox lets user type or click arrows, clamped to 1-10
            tk.Spinbox(row, from_=1, to=10, textvariable=var,
                       width=5, justify="center").pack(side="left", padx=8)

        # ── confirm / cancel buttons ──
        btn_row = tk.Frame(self)
        btn_row.pack(pady=12)
        tk.Button(btn_row, text="✅ Confirm", command=self.confirm,
                  bg="#4f8ef7", fg="white", relief="flat", padx=14, pady=5).pack(side="left", padx=6)
        tk.Button(btn_row, text="Cancel", command=self.destroy,
                  bg="#e8edf8", fg="#333", relief="flat", padx=14, pady=5).pack(side="left", padx=6)

        # center popup over parent window
        self.update_idletasks()
        x = parent.winfo_x() + (parent.winfo_width() // 2) - (self.winfo_width() // 2)
        y = parent.winfo_y() + (parent.winfo_height() // 2) - (self.winfo_height() // 2)
        self.geometry(f"+{x}+{y}")

    def confirm(self):
        # Validate all scores are within 1-10 before accepting
        scores = {}
        for name, var in self.score_vars.items():
            try:
                val = int(var.get())
                if not 1 <= val <= 10:
                    raise ValueError
                scores[name] = val
            except (ValueError, tk.TclError):
                messagebox.showerror("Invalid score",
                    f'Score for "{name}" must be a number between 1 and 10.')
                return
        self.result = scores
        self.destroy()


# ── MAIN APP ──────────────────────────────────────────────────────────────────

class GroupRandomizer:
    def __init__(self, window):
        self.window = window
        self.window.title("Group Randomizer")
        self.window.resizable(True, True)

        # Stores scores from the last time the score editor was confirmed.
        # Persists between mode switches so you don't re-enter scores each time.
        self.saved_scores = {}

        # Last generated groups for export.
        # Each item is a list of names (random) or (name, score) tuples (balanced).
        self.last_results = []
        self.last_mode = "random"

        self.build_ui()

    # ── UI SETUP ──────────────────────────────────────────────────────────────

    def build_ui(self):
        self.window.configure(bg="#f4f6ff")
        self.window.minsize(640, 560)

        # ── Title ──
        tk.Label(self.window, text="👥 Group Randomizer", font=("TkDefaultFont", 16, "bold"),
                 bg="#f4f6ff").pack(pady=(16, 0))
        tk.Label(self.window, text="Split participants into random or balanced groups",
                 font=("TkDefaultFont", 10), fg="#888", bg="#f4f6ff").pack(pady=(2, 12))

        # ── Save / Load bar ──
        bar = tk.Frame(self.window, bg="#f4f6ff")
        bar.pack(fill="x", padx=12, pady=6)
        tk.Button(bar, text="💾 Save List", command=self.save_list,
                  bg="#4f8ef7", fg="white", relief="flat", padx=10).pack(side="left", padx=(0, 6))
        tk.Button(bar, text="📂 Load List", command=self.load_list,
                  bg="#6c757d", fg="white", relief="flat", padx=10).pack(side="left")

        # ── Participants input ──
        tk.Label(self.window, text="Participants (one per line)",
                 font=("TkDefaultFont", 10, "bold"), bg="#f4f6ff").pack(anchor="w", padx=12)
        self.part_text = tk.Text(self.window, height=10, relief="solid", bd=1,
                                 font=("TkDefaultFont", 11))
        self.part_text.pack(fill="both", expand=True, padx=12, pady=(4, 8))
        self.part_text.insert("1.0", "Alice\nBob\nCarla\nDan\nEve\nFrank")

        # ── Options row: mode + group count ──
        options = tk.Frame(self.window, bg="#f4f6ff")
        options.pack(fill="x", padx=12, pady=4)

        # Mode selector — radio buttons let user pick one mode at a time.
        # self.mode_var holds the current selection: 'random' or 'balanced'.
        tk.Label(options, text="Mode:", font=("TkDefaultFont", 10, "bold"),
                 bg="#f4f6ff").pack(side="left", padx=(0, 6))
        self.mode_var = tk.StringVar(value="random")
        tk.Radiobutton(options, text="🎲 Random", variable=self.mode_var,
                       value="random", bg="#f4f6ff",
                       font=("TkDefaultFont", 10)).pack(side="left", padx=(0, 12))
        tk.Radiobutton(options, text="⚖️ Balanced", variable=self.mode_var,
                       value="balanced", bg="#f4f6ff",
                       font=("TkDefaultFont", 10)).pack(side="left", padx=(0, 20))

        # Number of groups input
        tk.Label(options, text="Number of groups:", font=("TkDefaultFont", 10, "bold"),
                 bg="#f4f6ff").pack(side="left", padx=(0, 6))
        self.group_count_var = tk.StringVar(value="3")
        tk.Entry(options, textvariable=self.group_count_var, width=5,
                 font=("TkDefaultFont", 11), justify="center",
                 relief="solid", bd=1).pack(side="left")

        # ── Action buttons ──
        btn_row = tk.Frame(self.window, bg="#f4f6ff")
        btn_row.pack(pady=8)
        tk.Button(btn_row, text="⚙️ Set Scores", command=self.open_score_editor,
                  bg="#6f42c1", fg="white", relief="flat", padx=12, pady=6).pack(side="left", padx=4)
        tk.Button(btn_row, text="👥 Form Groups", command=self.form_groups,
                  bg="#4f8ef7", fg="white", font=("TkDefaultFont", 12, "bold"),
                  relief="flat", padx=20, pady=6).pack(side="left", padx=4)
        tk.Button(btn_row, text="Clear Results", command=self.clear_all,
                  bg="#e8edf8", fg="#333", relief="flat", padx=12, pady=6).pack(side="left", padx=4)
        tk.Button(btn_row, text="📄 Export .txt", command=self.export_txt,
                  bg="#28a745", fg="white", relief="flat", padx=12, pady=6).pack(side="left", padx=4)
        tk.Button(btn_row, text="📊 Export .csv", command=self.export_csv,
                  bg="#28a745", fg="white", relief="flat", padx=12, pady=6).pack(side="left", padx=4)

        # ── Results table ──
        # Three columns: Group number, Members (comma separated), Total Score.
        # Total Score column shows '—' in random mode since scores aren't used.
        # In balanced mode it shows the sum of all member scores in that group.
        result_frame = tk.Frame(self.window, bg="#f4f6ff")
        result_frame.pack(fill="both", expand=False, padx=12, pady=6)

        cols_cfg = ("Group", "Members", "Total Score")
        self.tree = ttk.Treeview(result_frame, columns=cols_cfg, show="headings", height=8)
        self.tree.heading("Group", text="Group")
        self.tree.heading("Members", text="Members")
        self.tree.heading("Total Score", text="Total Score")
        self.tree.column("Group", width=70, anchor="center")
        self.tree.column("Members", width=380)
        self.tree.column("Total Score", width=90, anchor="center")

        scrollbar = ttk.Scrollbar(result_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        self.tree.tag_configure("odd", background="#ffffff")
        self.tree.tag_configure("even", background="#f0f4ff")

        # ── Status bar ──
        self.status = tk.Label(self.window, text="", fg="#888", bg="#f4f6ff",
                               font=("TkDefaultFont", 10))
        self.status.pack(pady=(0, 12))

    # ── SCORE EDITOR ──────────────────────────────────────────────────────────

    def open_score_editor(self):
        # Read current participant list first
        participants = self.get_participants()
        if not participants:
            messagebox.showwarning("No participants", "Add participants before setting scores.")
            return
        # Open popup, pass existing scores so they're preserved between opens
        editor = ScoreEditor(self.window, participants, self.saved_scores)
        self.window.wait_window(editor)  # block until popup closes
        if editor.result is not None:
            self.saved_scores = editor.result
            self.status.config(text=f"✅ Scores saved for {len(self.saved_scores)} participants.")

    # ── FORM GROUPS ───────────────────────────────────────────────────────────

    def form_groups(self):
        participants = self.get_participants()
        if not participants:
            messagebox.showerror("No participants", "Add at least one participant.")
            return

        # Validate group count
        try:
            group_count = int(self.group_count_var.get())
            if group_count < 2:
                raise ValueError
        except ValueError:
            messagebox.showerror("Invalid number", "Number of groups must be 2 or more.")
            return

        if group_count > len(participants):
            messagebox.showerror("Too many groups",
                f"You have {len(participants)} participants but asked for {group_count} groups.\n"
                "There must be at least 1 person per group.")
            return

        mode = self.mode_var.get()
        self.last_mode = mode

        if mode == "random":
            groups = random_split(participants, group_count)
            # Store as list of name lists
            self.last_results = groups

        else:  # balanced
            # Check all participants have scores set
            missing = [p for p in participants if p not in self.saved_scores]
            if missing:
                answer = messagebox.askyesno("Missing scores",
                    f"{len(missing)} participant(s) have no score set.\n"
                    "They will be assigned a default score of 5.\n\nContinue?")
                if not answer:
                    return
                for name in missing:
                    self.saved_scores[name] = 5

            students = [(name, self.saved_scores[name]) for name in participants]
            groups = balanced_split(students, group_count)
            self.last_results = groups

        # Clear old table rows and repopulate with new results.
        # Random mode: members are plain strings.
        # Balanced mode: members are (name, score) tuples, so we format them as 'name (score)'
        # and compute each group's total score for the last column.
        self.clear_results()
        for i, group in enumerate(groups):
            tag = "even" if i % 2 == 0 else "odd"
            if mode == "random":
                members_str = ", ".join(group)
                self.tree.insert("", "end", values=(f"Group {i+1}", members_str, "—"), tags=(tag,))
            else:
                members_str = ", ".join(f"{name} ({score})" for name, score in group)
                total = sum(score for _, score in group)
                self.tree.insert("", "end", values=(f"Group {i+1}", members_str, total), tags=(tag,))

        self.status.config(text=f"✅ {len(groups)} groups formed ({mode} mode).")

    # ── HELPERS ───────────────────────────────────────────────────────────────

    def get_participants(self):
        # Read participant text box, strip blanks
        return [l.strip() for l in self.part_text.get("1.0", "end").splitlines() if l.strip()]

    def clear_results(self):
        # Only clears the table display, not last_results (same pattern as lottery app)
        for row in self.tree.get_children():
            self.tree.delete(row)
        self.status.config(text="")

    def clear_all(self):
        # Full clear — wipes table and last_results. Export will ask to draw first after this.
        self.clear_results()
        self.last_results = []

    # ── SAVE / LOAD ───────────────────────────────────────────────────────────
    # JSON format: { "participants": [...], "scores": {...} }

    def save_list(self):
        participants = self.get_participants()
        if not participants:
            messagebox.showwarning("Nothing to save", "Participant list is empty.")
            return
        path = filedialog.asksaveasfilename(
            title="Save list",
            defaultextension=".json",
            filetypes=[("JSON files", "*.json"), ("All files", "*.*")]
        )
        if not path:
            return
        with open(path, "w", encoding="utf-8") as f:
            json.dump({"participants": participants, "scores": self.saved_scores}, f, indent=2)
        self.status.config(text=f"✅ Saved to: {path}")

    def load_list(self):
        path = filedialog.askopenfilename(
            title="Open a saved list",
            filetypes=[("JSON files", "*.json"), ("All files", "*.*")]
        )
        if not path:
            return
        try:
            with open(path, "r", encoding="utf-8") as f:
                data = json.load(f)
            participants = data.get("participants", [])
            self.part_text.delete("1.0", "end")
            self.part_text.insert("1.0", "\n".join(participants))
            self.saved_scores = data.get("scores", {})
            self.clear_results()
            self.status.config(text=f"📂 Loaded: {path}")
        except Exception as e:
            messagebox.showerror("Error", f"Could not load file:\n{e}")

    # ── EXPORT ────────────────────────────────────────────────────────────────

    def export_txt(self):
        # Writes a plain text file with one group per line.
        # Balanced mode includes total score per group in brackets.
        if not self.last_results:
            messagebox.showwarning("No results", "Form groups first before exporting.")
            return
        path = filedialog.asksaveasfilename(
            title="Export as text",
            defaultextension=".txt",
            filetypes=[("Text files", "*.txt"), ("All files", "*.*")]
        )
        if not path:
            return
        with open(path, "w", encoding="utf-8") as f:
            f.write(f"Group Randomizer Results ({self.last_mode} mode)\n")
            f.write("=" * 40 + "\n")
            for i, group in enumerate(self.last_results, 1):
                if self.last_mode == "random":
                    members_str = ", ".join(group)
                    f.write(f"Group {i}: {members_str}\n")
                else:
                    members_str = ", ".join(f"{name} ({score})" for name, score in group)
                    total = sum(score for _, score in group)
                    f.write(f"Group {i} [total: {total}]: {members_str}\n")
        self.status.config(text=f"📄 Exported to: {path}")

    def export_csv(self):
        # Writes a CSV file. Random mode has 2 columns, balanced has 3 (adds Total Score).
        # newline='' is required by Python's csv module to avoid extra blank lines.
        if not self.last_results:
            messagebox.showwarning("No results", "Form groups first before exporting.")
            return
        path = filedialog.asksaveasfilename(
            title="Export as CSV",
            defaultextension=".csv",
            filetypes=[("CSV files", "*.csv"), ("All files", "*.*")]
        )
        if not path:
            return
        with open(path, "w", newline="", encoding="utf-8") as f:
            writer = csv.writer(f)
            if self.last_mode == "random":
                writer.writerow(["Group", "Members"])
                for i, group in enumerate(self.last_results, 1):
                    writer.writerow([f"Group {i}", ", ".join(group)])
            else:
                writer.writerow(["Group", "Members", "Total Score"])
                for i, group in enumerate(self.last_results, 1):
                    members_str = ", ".join(f"{name} ({score})" for name, score in group)
                    total = sum(score for _, score in group)
                    writer.writerow([f"Group {i}", members_str, total])
        self.status.config(text=f"📊 Exported to: {path}")


# ── ENTRY POINT ───────────────────────────────────────────────────────────────

if __name__ == "__main__":
    main_window = tk.Tk()
    app = GroupRandomizer(main_window)
    main_window.mainloop()
