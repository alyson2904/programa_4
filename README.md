import tkinter as tk
from tkinter import ttk, messagebox, filedialog
from PIL import Image, ImageTk
import face_recognition
import cv2
import os
import numpy as np
import mysql.connector
from datetime import datetime, time
import time as time_module
import logging
import threading

# Configuración de logging (igual que antes)
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('asistencia.log'),
        logging.StreamHandler()
    ]
)

class App:
    def __init__(self, root):
        self.root = root
        self.root.title("Sistema de Reconocimiento Facial")
        self.root.geometry("1000x700")
        
        # Variables de estado
        self.db = None
        self.cap = None
        self.running = False
        self.current_frame = None
        
        # Conectar a la base de datos al iniciar
        self.conectar_db()
        
        # Crear pestañas
        self.notebook = ttk.Notebook(root)
        self.notebook.pack(fill=tk.BOTH, expand=True)
        
        # Pestaña de registro
        self.registro_tab = ttk.Frame(self.notebook)
        self.notebook.add(self.registro_tab, text="Registrar Persona")
        self.setup_registro_tab()
        
        # Pestaña de asistencia
        self.asistencia_tab = ttk.Frame(self.notebook)
        self.notebook.add(self.asistencia_tab, text="Tomar Asistencia")
        self.setup_asistencia_tab()
        
        # Pestaña de reportes
        self.reportes_tab = ttk.Frame(self.notebook)
        self.notebook.add(self.reportes_tab, text="Reportes")
        self.setup_reportes_tab()
        
        # Manejar cierre de la ventana
        self.root.protocol("WM_DELETE_WINDOW", self.on_close)
    
    def conectar_db(self):
        """Establece conexión con la base de datos"""
        try:
            self.db = mysql.connector.connect(
                host="localhost",
                user="root",
                password="",
                database="rostros",
                autocommit=True
            )
            logging.info("Conexión a la base de datos establecida")
            self.inicializar_db()
        except mysql.connector.Error as err:
            logging.error(f"Error al conectar a la base de datos: {err}")
            messagebox.showerror("Error", "No se pudo conectar a la base de datos")
    
    def inicializar_db(self):
        """Crea las tablas necesarias si no existen"""
        if self.db is None:
            return False
        
        try:
            with self.db.cursor() as cursor:
                cursor.execute("""
                    CREATE TABLE IF NOT EXISTS personas (
                        id INT AUTO_INCREMENT PRIMARY KEY,
                        nombre VARCHAR(100) NOT NULL,
                        imagen_nombre VARCHAR(255) NOT NULL,
                        fecha_registro TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                    ) ENGINE=InnoDB
                """)
                
                cursor.execute("""
                    CREATE TABLE IF NOT EXISTS asistencias (
                        id INT AUTO_INCREMENT PRIMARY KEY,
                        persona_id INT NOT NULL,
                        fecha DATE NOT NULL,
                        hora TIME NOT NULL,
                        estado VARCHAR(10) NOT NULL,
                        UNIQUE KEY unique_asistencia (persona_id, fecha),
                        FOREIGN KEY (persona_id) REFERENCES personas(id)
                    ) ENGINE=InnoDB
                """)
                
                self.db.commit()
                logging.info("Tablas inicializadas correctamente")
                return True
        except mysql.connector.Error as err:
            logging.error(f"Error al inicializar la base de datos: {err}")
            return False
    
    def setup_registro_tab(self):
        """Configura la interfaz de registro de personas"""
        frame = ttk.LabelFrame(self.registro_tab, text="Registrar Nueva Persona")
        frame.pack(pady=10, padx=10, fill=tk.BOTH, expand=True)
        
        # Campo de nombre
        ttk.Label(frame, text="Nombre:").grid(row=0, column=0, padx=5, pady=5, sticky=tk.W)
        self.nombre_entry = ttk.Entry(frame, width=30)
        self.nombre_entry.grid(row=0, column=1, padx=5, pady=5)
        
        # Botón para capturar imagen
        self.capturar_btn = ttk.Button(frame, text="Iniciar Captura", command=self.iniciar_captura)
        self.capturar_btn.grid(row=1, column=0, columnspan=2, pady=10)
        
        # Área para mostrar la cámara
        self.camera_label = ttk.Label(frame)
        self.camera_label.grid(row=2, column=0, columnspan=2, pady=10)
        
        # Botón para guardar registro
        self.guardar_btn = ttk.Button(frame, text="Guardar Registro", state=tk.DISABLED, command=self.guardar_registro)
        self.guardar_btn.grid(row=3, column=0, columnspan=2, pady=10)
        
        # Variables para el registro
        self.face_encoding = None
        self.imagen_nombre = None
    
    def setup_asistencia_tab(self):
        """Configura la interfaz para tomar asistencia"""
        frame = ttk.LabelFrame(self.asistencia_tab, text="Tomar Asistencia")
        frame.pack(pady=10, padx=10, fill=tk.BOTH, expand=True)
        
        # Botón para iniciar/parar reconocimiento
        self.reconocer_btn = ttk.Button(frame, text="Iniciar Reconocimiento", command=self.toggle_reconocimiento)
        self.reconocer_btn.pack(pady=10)
        
        # Área para mostrar la cámara
        self.asistencia_camera_label = ttk.Label(frame)
        self.asistencia_camera_label.pack(pady=10)
        
        # Lista de asistencias del día
        columns = ("nombre", "hora", "estado")
        self.asistencias_tree = ttk.Treeview(frame, columns=columns, show="headings")
        
        self.asistencias_tree.heading("nombre", text="Nombre")
        self.asistencias_tree.heading("hora", text="Hora")
        self.asistencias_tree.heading("estado", text="Estado")
        
        self.asistencias_tree.pack(fill=tk.BOTH, expand=True, pady=10)
        
        # Actualizar lista de asistencias
        self.actualizar_lista_asistencias()
    
    def setup_reportes_tab(self):
        """Configura la interfaz para generar reportes"""
        frame = ttk.LabelFrame(self.reportes_tab, text="Reportes de Asistencia")
        frame.pack(pady=10, padx=10, fill=tk.BOTH, expand=True)
        
        # Filtros de fecha
        ttk.Label(frame, text="Fecha inicial:").grid(row=0, column=0, padx=5, pady=5)
        self.fecha_inicio_entry = ttk.Entry(frame)
        self.fecha_inicio_entry.grid(row=0, column=1, padx=5, pady=5)
        
        ttk.Label(frame, text="Fecha final:").grid(row=1, column=0, padx=5, pady=5)
        self.fecha_fin_entry = ttk.Entry(frame)
        self.fecha_fin_entry.grid(row=1, column=1, padx=5, pady=5)
        
        # Botón para generar reporte
        ttk.Button(frame, text="Generar Reporte", command=self.generar_reporte).grid(row=2, column=0, columnspan=2, pady=10)
        
        # Área para mostrar el reporte
        columns = ("fecha", "nombre", "hora", "estado")
        self.reportes_tree = ttk.Treeview(frame, columns=columns, show="headings")
        
        self.reportes_tree.heading("fecha", text="Fecha")
        self.reportes_tree.heading("nombre", text="Nombre")
        self.reportes_tree.heading("hora", text="Hora")
        self.reportes_tree.heading("estado", text="Estado")
        
        self.reportes_tree.grid(row=3, column=0, columnspan=2, sticky=tk.NSEW, pady=10)
        
        # Configurar expansión
        frame.grid_rowconfigure(3, weight=1)
        frame.grid_columnconfigure(0, weight=1)
        frame.grid_columnconfigure(1, weight=1)
    
    def iniciar_captura(self):
        """Inicia la captura de video para registrar una nueva persona"""
        nombre = self.nombre_entry.get().strip()
        if not nombre:
            messagebox.showwarning("Advertencia", "Por favor ingrese un nombre")
            return
        
        if self.cap is not None:
            self.cap.release()
        
        self.cap = cv2.VideoCapture(0, cv2.CAP_DSHOW)
        if not self.cap.isOpened():
            messagebox.showerror("Error", "No se pudo acceder a la cámara")
            return
        
        self.capturar_btn.config(state=tk.DISABLED)
        self.guardar_btn.config(state=tk.DISABLED)
        self.running = True
        
        # Hilo para capturar video
        threading.Thread(target=self.actualizar_captura_registro, daemon=True).start()
    
    def actualizar_captura_registro(self):
        """Actualiza la imagen de la cámara para registro"""
        while self.running:
            ret, frame = self.cap.read()
            if not ret:
                break
            
            # Convertir a RGB para face_recognition
            rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            
            # Detectar rostros
            face_locations = face_recognition.face_locations(rgb_frame, model="hog")
            
            if face_locations:
                # Obtener codificación facial
                self.face_encoding = face_recognition.face_encodings(rgb_frame, face_locations)[0]
                
                # Dibujar rectángulo alrededor del rostro
                top, right, bottom, left = face_locations[0]
                cv2.rectangle(frame, (left, top), (right, bottom), (0, 255, 0), 2)
                
                # Habilitar botón de guardar
                self.guardar_btn.config(state=tk.NORMAL)
            
            # Mostrar frame en la interfaz
            frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            img = Image.fromarray(frame)
            imgtk = ImageTk.PhotoImage(image=img)
            
            self.camera_label.imgtk = imgtk
            self.camera_label.configure(image=imgtk)
        
        if self.cap is not None:
            self.cap.release()
            self.cap = None
    
    def guardar_registro(self):
        """Guarda el registro de la nueva persona"""
        nombre = self.nombre_entry.get().strip()
        if not nombre or self.face_encoding is None:
            return
        
        # Generar nombre único para la imagen
        timestamp = datetime.now().strftime('%Y%m%d%H%M%S')
        self.imagen_nombre = f"{nombre}_{timestamp}.jpg"
        directorio = "rostros_conocidos"
        os.makedirs(directorio, exist_ok=True)
        imagen_path = os.path.join(directorio, self.imagen_nombre)
        
        # Guardar imagen
        ret, frame = self.cap.read()
        if ret:
            cv2.imwrite(imagen_path, frame)
        
        # Registrar en la base de datos
        try:
            with self.db.cursor() as cursor:
                cursor.execute(
                    "INSERT INTO personas (nombre, imagen_nombre) VALUES (%s, %s)",
                    (nombre, self.imagen_nombre)
                )
                self.db.commit()
                logging.info(f"Persona registrada: {nombre} con imagen {self.imagen_nombre}")
                messagebox.showinfo("Éxito", f"Persona {nombre} registrada correctamente")
                
                # Limpiar formulario
                self.nombre_entry.delete(0, tk.END)
                self.guardar_btn.config(state=tk.DISABLED)
                self.capturar_btn.config(state=tk.NORMAL)
                self.running = False
                
        except mysql.connector.Error as err:
            logging.error(f"Error al registrar persona: {err}")
            messagebox.showerror("Error", f"No se pudo registrar a la persona: {err}")
    
    def toggle_reconocimiento(self):
        """Inicia o detiene el reconocimiento facial"""
        if not self.running:
            # Iniciar reconocimiento
            if self.cap is not None:
                self.cap.release()
            
            self.cap = cv2.VideoCapture(0, cv2.CAP_DSHOW)
            if not self.cap.isOpened():
                messagebox.showerror("Error", "No se pudo acceder a la cámara")
                return
            
            self.running = True
            self.reconocer_btn.config(text="Detener Reconocimiento")
            
            # Cargar rostros conocidos
            self.cargar_rostros_conocidos()
            
            # Iniciar hilo de reconocimiento
            threading.Thread(target=self.actualizar_reconocimiento, daemon=True).start()
        else:
            # Detener reconocimiento
            self.running = False
            self.reconocer_btn.config(text="Iniciar Reconocimiento")
            if self.cap is not None:
                self.cap.release()
                self.cap = None
    
    def cargar_rostros_conocidos(self):
        """Carga los rostros conocidos de la base de datos"""
        self.rostros_conocidos = []
        self.nombres_conocidos = []
        self.ids_conocidos = []
        
        try:
            with self.db.cursor(dictionary=True) as cursor:
                cursor.execute("SELECT id, nombre, imagen_nombre FROM personas")
                personas = cursor.fetchall()

            for persona in personas:
                imagen_path = os.path.join("rostros_conocidos", persona['imagen_nombre'])
                if os.path.exists(imagen_path):
                    try:
                        imagen = face_recognition.load_image_file(imagen_path)
                        codificaciones = face_recognition.face_encodings(imagen)
                        if codificaciones:
                            self.rostros_conocidos.append(codificaciones[0])
                            self.nombres_conocidos.append(persona['nombre'])
                            self.ids_conocidos.append(persona['id'])
                    except Exception as e:
                        logging.error(f"Error procesando imagen {persona['imagen_nombre']}: {e}")
            
            if not self.rostros_conocidos:
                messagebox.showwarning("Advertencia", "No hay rostros registrados en el sistema")
                self.running = False
                self.reconocer_btn.config(text="Iniciar Reconocimiento")
        
        except mysql.connector.Error as err:
            logging.error(f"Error al cargar rostros conocidos: {err}")
            messagebox.showerror("Error", "No se pudieron cargar los rostros registrados")
            self.running = False
            self.reconocer_btn.config(text="Iniciar Reconocimiento")
    
    def actualizar_reconocimiento(self):
        """Realiza el reconocimiento facial y actualiza la interfaz"""
        while self.running and self.cap is not None:
            ret, frame = self.cap.read()
            if not ret:
                break
            
            # Reducir tamaño para mayor eficiencia
            small_frame = cv2.resize(frame, (0, 0), fx=0.25, fy=0.25)
            rgb_small_frame = cv2.cvtColor(small_frame, cv2.COLOR_BGR2RGB)
            
            # Detectar rostros
            face_locations = face_recognition.face_locations(rgb_small_frame, model="hog")
            face_encodings = face_recognition.face_encodings(rgb_small_frame, face_locations)

            for (top, right, bottom, left), face_encoding in zip(face_locations, face_encodings):
                # Comparar con rostros conocidos
                coincidencias = face_recognition.compare_faces(self.rostros_conocidos, face_encoding, tolerance=0.6)
                nombre = "Desconocido"
                estado = ""
                color = (0, 0, 255)  # Rojo por defecto

                if True in coincidencias:
                    # Obtener el mejor match
                    indice = np.argmin(face_recognition.face_distance(self.rostros_conocidos, face_encoding))
                    nombre = self.nombres_conocidos[indice]
                    persona_id = self.ids_conocidos[indice]
                    ahora = datetime.now()
                    
                    # Verificar si ya registró asistencia hoy
                    try:
                        with self.db.cursor(dictionary=True) as cursor:
                            cursor.execute(
                                "SELECT hora, estado FROM asistencias WHERE persona_id = %s AND fecha = %s",
                                (persona_id, ahora.date())
                            )
                            registro = cursor.fetchone()

                        if registro:
                            # Ya registrado
                            estado = f"Registrado a las {registro['hora']}"
                            color = (255, 255, 0)  # Amarillo
                        else:
                            # Registrar nueva asistencia
                            estado = "Temprano" if ahora.time() <= time(8, 0, 0) else "Tarde"
                            color = (0, 255, 0)  # Verde
                            
                            try:
                                with self.db.cursor() as cursor:
                                    cursor.execute(
                                        "INSERT INTO asistencias (persona_id, fecha, hora, estado) VALUES (%s, %s, %s, %s)",
                                        (persona_id, ahora.date(), ahora.time(), estado)
                                    )
                                    self.db.commit()
                                    logging.info(f"Asistencia registrada: {nombre} - {estado} a las {ahora.time()}")
                                    self.actualizar_lista_asistencias()
                            except mysql.connector.Error as err:
                                if err.errno != 1062:  # Ignorar duplicados
                                    logging.error(f"Error al registrar asistencia: {err}")
                    except mysql.connector.Error as err:
                        logging.error(f"Error al verificar asistencia: {err}")

                # Escalar coordenadas al tamaño original
                top *= 4; right *= 4; bottom *= 4; left *= 4
                
                # Dibujar rectángulo y etiqueta
                cv2.rectangle(frame, (left, top), (right, bottom), color, 2)
                cv2.rectangle(frame, (left, bottom - 35), (right, bottom), color, cv2.FILLED)
                cv2.putText(frame, f"{nombre} {estado}", (left + 6, bottom - 6), 
                            cv2.FONT_HERSHEY_DUPLEX, 0.8, (255, 255, 255), 1)

            # Mostrar frame en la interfaz
            frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            img = Image.fromarray(frame)
            imgtk = ImageTk.PhotoImage(image=img)
            
            self.asistencia_camera_label.imgtk = imgtk
            self.asistencia_camera_label.configure(image=imgtk)
        
        if self.cap is not None:
            self.cap.release()
            self.cap = None
    
    def actualizar_lista_asistencias(self):
        """Actualiza la lista de asistencias del día actual"""
        try:
            with self.db.cursor(dictionary=True) as cursor:
                cursor.execute("""
                    SELECT p.nombre, a.hora, a.estado 
                    FROM asistencias a
                    JOIN personas p ON a.persona_id = p.id
                    WHERE a.fecha = %s
                    ORDER BY a.hora
                """, (datetime.now().date(),))
                asistencias = cursor.fetchall()
            
            # Limpiar lista actual
            for item in self.asistencias_tree.get_children():
                self.asistencias_tree.delete(item)
            
            # Agregar nuevas asistencias
            for asistencia in asistencias:
                self.asistencias_tree.insert("", tk.END, values=(
                    asistencia['nombre'],
                    str(asistencia['hora']),
                    asistencia['estado']
                ))
        except mysql.connector.Error as err:
            logging.error(f"Error al obtener asistencias: {err}")
    
    def generar_reporte(self):
        """Genera un reporte de asistencias según los filtros"""
        fecha_inicio = self.fecha_inicio_entry.get().strip()
        fecha_fin = self.fecha_fin_entry.get().strip()
        
        try:
            query = """
                SELECT a.fecha, p.nombre, a.hora, a.estado 
                FROM asistencias a
                JOIN personas p ON a.persona_id = p.id
            """
            params = []
            
            if fecha_inicio or fecha_fin:
                query += " WHERE "
                conditions = []
                
                if fecha_inicio:
                    conditions.append("a.fecha >= %s")
                    params.append(fecha_inicio)
                
                if fecha_fin:
                    conditions.append("a.fecha <= %s")
                    params.append(fecha_fin)
                
                query += " AND ".join(conditions)
            
            query += " ORDER BY a.fecha, a.hora"
            
            with self.db.cursor(dictionary=True) as cursor:
                cursor.execute(query, tuple(params))
                reporte = cursor.fetchall()
            
            # Limpiar lista actual
            for item in self.reportes_tree.get_children():
                self.reportes_tree.delete(item)
            
            # Agregar datos al reporte
            for fila in reporte:
                self.reportes_tree.insert("", tk.END, values=(
                    fila['fecha'].strftime('%Y-%m-%d'),
                    fila['nombre'],
                    str(fila['hora']),
                    fila['estado']
                ))
        except mysql.connector.Error as err:
            logging.error(f"Error al generar reporte: {err}")
            messagebox.showerror("Error", f"No se pudo generar el reporte: {err}")
    
    def on_close(self):
        """Maneja el cierre de la aplicación"""
        self.running = False
        if self.cap is not None:
            self.cap.release()
        if self.db is not None and self.db.is_connected():
            self.db.close()
        self.root.destroy()

if __name__ == "__main__":
    root = tk.Tk()
    app = App(root)
    root.mainloop()
