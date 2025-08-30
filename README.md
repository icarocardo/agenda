# agenda
nós fizemos uma agenda de  tarefa contendo o nome e a descrição. nela há a possibilidade de colocar data inicial e final. E a situação da tarefa se foi ou não feita
import sqlite3
import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime

# ---------------- BANCO DE DADOS ----------------
conn = sqlite3.connect("agenda_tarefas.db")
cur = conn.cursor()
cur.execute("""
CREATE TABLE IF NOT EXISTS tarefas (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nome TEXT NOT NULL,
    descricao TEXT,
    data_inicial TEXT NOT NULL,   -- armazenado como string DD/MM/AAAA
    data_final   TEXT NOT NULL,
    tipo TEXT,
    situacao TEXT NOT NULL CHECK (situacao IN ('Não feito','Feito'))
)
""")
conn.commit()

# ---------------- FUNÇÕES ÚTEIS ----------------
DATE_FMT = "%d/%m/%Y"

def validar_data(s: str) -> bool:
    try:
        datetime.strptime(s, DATE_FMT)
        return True
    except ValueError:
        return False

def limpar_campos():
    entry_nome.delete(0, tk.END)
    txt_desc.delete("1.0", tk.END)
    entry_ini.delete(0, tk.END)
    entry_fim.delete(0, tk.END)
    combo_tipo.set("")
    combo_sit.set("Não feito")
    tree.selection_remove(tree.selection())

def preencher_form_da_selecao(_evt=None):
    sel = tree.selection()
    if not sel:
        return
    vals = tree.item(sel[0], "values")
    entry_nome.delete(0, tk.END); entry_nome.insert(0, vals[1])
    txt_desc.delete("1.0", tk.END); txt_desc.insert("1.0", vals[2])
    entry_ini.delete(0, tk.END); entry_ini.insert(0, vals[3])
    entry_fim.delete(0, tk.END); entry_fim.insert(0, vals[4])
    combo_tipo.set(vals[5])
    combo_sit.set(vals[6])

def atualizar_lista(filtro_sit=None, termo_busca=None):
    for i in tree.get_children():
        tree.delete(i)

    sql = "SELECT * FROM tarefas"
    params = []
    where = []
    if filtro_sit:
        where.append("situacao = ?"); params.append(filtro_sit)
    if termo_busca:
        where.append("(nome LIKE ? OR descricao LIKE ?)")
        params += [f"%{termo_busca}%", f"%{termo_busca}%"]
    if where:
        sql += " WHERE " + " AND ".join(where)
    sql += " ORDER BY date(substr(data_inicial,7,4)||'-'||substr(data_inicial,4,2)||'-'||substr(data_inicial,1,2)) ASC"

    for row in cur.execute(sql, params):
        tree.insert("", "end", values=row)

def adicionar_tarefa():
    nome = entry_nome.get().strip()
    desc = txt_desc.get("1.0", tk.END).strip()
    di = entry_ini.get().strip()
    df = entry_fim.get().strip()
    tipo = combo_tipo.get().strip()
    sit = combo_sit.get().strip() or "Não feito"

    if not nome or not di or not df:
        messagebox.showwarning("Campos obrigatórios", "Preencha Nome, Data inicial e Data final.")
        return
    if not validar_data(di) or not validar_data(df):
        messagebox.showwarning("Data inválida", "Use o formato DD/MM/AAAA.")
        return
    if datetime.strptime(df, DATE_FMT) < datetime.strptime(di, DATE_FMT):
        messagebox.showwarning("Datas", "Data final não pode ser anterior à data inicial.")
        return

    cur.execute("""INSERT INTO tarefas (nome, descricao, data_inicial, data_final, tipo, situacao)
                   VALUES (?, ?, ?, ?, ?, ?)""",
                (nome, desc, di, df, tipo, sit or "Não feito"))
    conn.commit()
    atualizar_lista()
    limpar_campos()
    messagebox.showinfo("Sucesso", "Tarefa adicionada!")

def editar_tarefa():
    sel = tree.selection()
    if not sel:
        messagebox.showwarning("Seleção", "Selecione uma tarefa para editar.")
        return
    tarefa_id = tree.item(sel[0], "values")[0]

    nome = entry_nome.get().strip()
    desc = txt_desc.get("1.0", tk.END).strip()
    di = entry_ini.get().strip()
    df = entry_fim.get().strip()
    tipo = combo_tipo.get().strip()
    sit = combo_sit.get().strip()

    if not nome or not di or not df:
        messagebox.showwarning("Campos obrigatórios", "Preencha Nome, Data inicial e Data final.")
        return
    if not validar_data(di) or not validar_data(df):
        messagebox.showwarning("Data inválida", "Use o formato DD/MM/AAAA.")
        return
    if datetime.strptime(df, DATE_FMT) < datetime.strptime(di, DATE_FMT):
        messagebox.showwarning("Datas", "Data final não pode ser anterior à data inicial.")
        return

    cur.execute("""UPDATE tarefas
                   SET nome=?, descricao=?, data_inicial=?, data_final=?, tipo=?, situacao=?
                   WHERE id=?""",
                (nome, desc, di, df, tipo, sit, tarefa_id))
    conn.commit()
    atualizar_lista()
    messagebox.showinfo("Sucesso", "Tarefa atualizada!")

def excluir_tarefa():
    sel = tree.selection()
    if not sel:
        messagebox.showwarning("Seleção", "Selecione uma tarefa para excluir.")
        return
    tarefa = tree.item(sel[0], "values")
    if messagebox.askyesno("Confirmar", f"Excluir a tarefa '{tarefa[1]}'?"):
        cur.execute("DELETE FROM tarefas WHERE id = ?", (tarefa[0],))
        conn.commit()
        atualizar_lista()
        limpar_campos()

def alternar_situacao():
    sel = tree.selection()
    if not sel:
        messagebox.showwarning("Seleção", "Selecione uma tarefa.")
        return
    vals = tree.item(sel[0], "values")
    tarefa_id, situacao_atual = vals[0], vals[6]
    nova = "Feito" if situacao_atual == "Não feito" else "Não feito"
    cur.execute("UPDATE tarefas SET situacao=? WHERE id=?", (nova, tarefa_id))
    conn.commit()
    atualizar_lista()

def buscar():
    termo = entry_busca.get().strip()
    filtro = combo_filtro.get()
    filtro_sit = None if filtro == "Todas" else filtro
    atualizar_lista(filtro_sit=filtro_sit, termo_busca=termo)

# ---------------- INTERFACE ----------------
root = tk.Tk()
root.title("Agenda de Tarefas")
root.geometry("900x600")

# --- Formulário ---
frm = ttk.LabelFrame(root, text="Tarefa")
frm.pack(fill="x", padx=10, pady=8)

ttk.Label(frm, text="Nome*").grid(row=0, column=0, sticky="w", padx=5, pady=4)
entry_nome = ttk.Entry(frm, width=40); entry_nome.grid(row=0, column=1, sticky="w", padx=5)

ttk.Label(frm, text="Tipo").grid(row=0, column=2, sticky="w", padx=5)
combo_tipo = ttk.Combobox(frm, values=["Pessoal","Trabalho","Estudos","Outro"], width=20)
combo_tipo.grid(row=0, column=3, sticky="w", padx=5)

ttk.Label(frm, text="Situação").grid(row=0, column=4, sticky="w", padx=5)
combo_sit = ttk.Combobox(frm, values=["Não feito","Feito"], width=12, state="readonly")
combo_sit.set("Não feito"); combo_sit.grid(row=0, column=5, sticky="w", padx=5)

ttk.Label(frm, text="Descrição").grid(row=1, column=0, sticky="nw", padx=5)
txt_desc = tk.Text(frm, width=60, height=4)
txt_desc.grid(row=1, column=1, columnspan=5, sticky="we", padx=5, pady=4)

ttk.Label(frm, text="Data inicial* (DD/MM/AAAA)").grid(row=2, column=0, sticky="w", padx=5)
entry_ini = ttk.Entry(frm, width=18); entry_ini.grid(row=2, column=1, sticky="w", padx=5)

ttk.Label(frm, text="Data final* (DD/MM/AAAA)").grid(row=2, column=2, sticky="w", padx=5)
entry_fim = ttk.Entry(frm, width=18); entry_fim.grid(row=2, column=3, sticky="w", padx=5)

btn_add = ttk.Button(frm, text="Adicionar", command=adicionar_tarefa); btn_add.grid(row=3, column=1, pady=6, sticky="w")
btn_edit = ttk.Button(frm, text="Salvar edição", command=editar_tarefa); btn_edit.grid(row=3, column=2, pady=6, sticky="w")
btn_toggle = ttk.Button(frm, text="Alternar situação", command=alternar_situacao); btn_toggle.grid(row=3, column=3, pady=6, sticky="w")
btn_clear = ttk.Button(frm, text="Limpar", command=limpar_campos); btn_clear.grid(row=3, column=4, pady=6, sticky="w")

# --- Filtro / Busca ---
filtro = ttk.LabelFrame(root, text="Filtros")
filtro.pack(fill="x", padx=10, pady=4)
ttk.Label(filtro, text="Buscar (nome/descrição):").grid(row=0, column=0, padx=5, pady=4)
entry_busca = ttk.Entry(filtro, width=40); entry_busca.grid(row=0, column=1, padx=5)
ttk.Label(filtro, text="Situação:").grid(row=0, column=2, padx=5)
combo_filtro = ttk.Combobox(filtro, values=["Todas","Não feito","Feito"], width=12, state="readonly")
combo_filtro.set("Todas"); combo_filtro.grid(row=0, column=3, padx=5)
ttk.Button(filtro, text="Aplicar", command=buscar).grid(row=0, column=4, padx=5)
ttk.Button(filtro, text="Mostrar todas", command=lambda: atualizar_lista()).grid(row=0, column=5, padx=5)

# --- Tabela ---
cols = ("ID","Nome","Descrição","Data inicial","Data final","Tipo","Situação")
tree = ttk.Treeview(root, columns=cols, show="headings", height=14)
for c in cols:
    tree.heading(c, text=c)
tree.column("ID", width=50, anchor="center")
tree.column("Nome", width=180)
tree.column("Descrição", width=260)
tree.column("Data inicial", width=110, anchor="center")
tree.column("Data final", width=110, anchor="center")
tree.column("Tipo", width=110, anchor="center")
tree.column("Situação", width=90, anchor="center")
tree.pack(fill="both", expand=True, padx=10, pady=6)

tree.bind("<<TreeviewSelect>>", preencher_form_da_selecao)

# --- Botões inferiores ---
frm_bottom = ttk.Frame(root); frm_bottom.pack(fill="x", padx=10, pady=4)
btn_del = ttk.Button(frm_bottom, text="Excluir selecionada", command=excluir_tarefa)
btn_del.pack(side="right")

# Inicializa lista
atualizar_lista()

root.mainloop()
conn.close() 
