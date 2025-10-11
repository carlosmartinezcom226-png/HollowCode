"""Sistema de reservación de salas """
from datetime import datetime, date, timedelta
import re
import json
import os

clientes = {}
salas = {}
reservas = []
contador_cliente = 0
contador_sala = 0
contador_reserva = 0

TURNOS = {
    '1': 'Matutino',
    '2': 'Vespertino',
    '3': 'Nocturno'
}

ARCHIVO_ESTADO = "estado_sistema.json"

def siguiente_id_cliente():
    """Genera y devuelve un nuevo ID único para un cliente."""
    global contador_cliente
    contador_cliente += 1
    return f"C{contador_cliente:03d}"

def siguiente_id_sala():
    """Genera y devuelve un nuevo ID único para una sala."""
    global contador_sala
    contador_sala += 1
    return f"S{contador_sala:03d}"

def siguiente_folio_reserva():
    """Genera y devuelve un nuevo folio único para una reservación."""
    global contador_reserva
    contador_reserva += 1
    return f"R{contador_reserva:03d}"

def convertir_fecha(texto):
    """Convierte una cadena de texto con formato DD/MM/YYYY en un objeto date.
     Retorna None si el formato no es válido."""
    try:
        return datetime.strptime(texto.strip(), "%d/%m/%Y").date()
    except Exception:
        return None

def fecha_valida(fecha_objetivo):
    """Verifica si la fecha es al menos dos días posterior al día actual."""
    hoy = date.today()
    minimo = hoy + timedelta(days=2)
    return fecha_objetivo >= minimo

def es_nombre_valido(texto):
    """Verifica que el texto contenga solo letras y espacios (acentos y ñ incluidos)."""
    return bool(re.fullmatch(r"[A-Za-zÁÉÍÓÚáéíóúÑñ ]+", texto)) and texto.strip() != ""

def guardar_estado():
    """Guarda todo el estado actual del sistema (clientes, salas, reservas y contadores)
     en un archivo JSON para su posterior recuperación."""
    estado = {
        'clientes': clientes,
        'salas': salas,
        'reservas': [
            {
                'folio': r['folio'],
                'cliente_id': r['cliente_id'],
                'sala_id': r['sala_id'],
                'fecha': r['fecha'].isoformat(),
                'turno': r['turno'],
                'evento': r['evento']
            } for r in reservas
        ],
        'contador_cliente': contador_cliente,
        'contador_sala': contador_sala,
        'contador_reserva': contador_reserva
    }
    with open(ARCHIVO_ESTADO, 'w', encoding='utf-8') as f:
        json.dump(estado, f, ensure_ascii=False, indent=2)
    print("Estado guardado correctamente en", ARCHIVO_ESTADO)

def cargar_estado():
    """Carga los datos previamente guardados desde el archivo JSON si existe
    Restaura clientes, salas, reservas y contadores."""
    global clientes, salas, reservas, contador_cliente, contador_sala, contador_reserva
    if not os.path.exists(ARCHIVO_ESTADO):
        print("No se encontró un estado previo. Iniciando con estado vacío.")
        return False
    try:
        with open(ARCHIVO_ESTADO, 'r', encoding='utf-8') as f:
            estado = json.load(f)
        clientes = estado.get('clientes', {})
        salas = estado.get('salas', {})
        reservas_raw = estado.get('reservas', [])
        reservas.clear()
        for r in reservas_raw:
            reservas.append({
                'folio': r['folio'],
                'cliente_id': r['cliente_id'],
                'sala_id': r['sala_id'],
                'fecha': datetime.fromisoformat(r['fecha']).date(),
                'turno': r['turno'],
                'evento': r['evento']
            })
        contador_cliente = estado.get('contador_cliente', 0)
        contador_sala = estado.get('contador_sala', 0)
        contador_reserva = estado.get('contador_reserva', 0)
        print("Estado previo cargado desde", ARCHIVO_ESTADO)
        return True
    except Exception as e:
        print("Error al cargar el estado:", str(e))
        print("Iniciando con estado vacío.")
        return False

def registrar_cliente():
    """Permite registrar un nuevo cliente solicitando nombre y apellidos válidos."""
    print("\n--- Registrar nuevo cliente ---")
    while True:
        nombre = input("Nombre: ").strip()
        if nombre.upper() == 'C':
            print("Operación cancelada.")
            return
        if not nombre or not es_nombre_valido(nombre):
            print("El nombre solo puede contener letras y espacios.")
            continue
        break

    while True:
        apellidos = input("Apellidos: ").strip()
        if apellidos.upper() == 'C':
            print("Operación cancelada.")
            return
        if not apellidos or not es_nombre_valido(apellidos):
            print("Los apellidos solo pueden contener letras y espacios.")
            continue
        break

    clave = siguiente_id_cliente()
    clientes[clave] = {'nombre': nombre, 'apellidos': apellidos}
    print(f"Cliente registrado con clave: {clave}\n")

def mostrar_clientes():
    """Muestra una lista ordenada de todos los clientes registrados."""
    if not clientes:
        print("\nNo hay clientes registrados.\n")
        return
    print("\nClientes registrados:")
    print(f"{'Clave':6} | {'Apellidos':25} | {'Nombre':25}")
    print("-" * 64)
    for clave, c in sorted(clientes.items(), key=lambda x: (x[1]['apellidos'].lower(), x[1]['nombre'].lower())):
        print(f"{clave:6} | {c['apellidos'][:25]:25} | {c['nombre'][:25]:25}")
    print()

def registrar_sala():
    """Permite registrar una nueva sala solicitando su nombre y cupo máximo."""
    print("\n--- Registrar nueva sala ---")
    while True:
        nombre = input("Nombre de la sala: ").strip()
        if nombre.upper() == 'C':
            print("Operación cancelada.")
            return
        if not re.fullmatch(r"[A-Za-zÁÉÍÓÚáéíóúÑñ0-9 ]+", nombre):
            print("El nombre de la sala solo puede contener letras, números y espacios.")
            continue
        break

    while True:
        cupo_txt = input("Cupo (número entero): ").strip()
        if cupo_txt.upper() == 'C':
            print("Operación cancelada.")
            return
        if not cupo_txt.isdigit() or int(cupo_txt) <= 0:
            print("Debe ingresar un número entero positivo.")
            continue
        cupo = int(cupo_txt)
        break

    clave = siguiente_id_sala()
    salas[clave] = {'nombre': nombre, 'cupo': cupo}
    print(f"Sala registrada con clave: {clave}\n")

def mostrar_salas():
    """Muestra una lista de las salas registradas con su nombre y capacidad."""
    if not salas:
        print("\nNo hay salas registradas.\n")
        return
    print("\nSalas registradas:")
    print(f"{'Clave':6} | {'Nombre':25} | {'Cupo':4}")
    print("-" * 40)
    for clave, s in salas.items():
        print(f"{clave:6} | {s['nombre'][:25]:25} | {str(s['cupo']):4}")
    print()

def salas_disponibles(fecha):
    """Devuelve una lista de turnos ocupados para una sala en una fecha específica."""
    disponibles = {}
    for clave_sala, s in salas.items():
        turnos_libres = set(TURNOS.keys())
        for res in reservas:
            if res['sala_id'] == clave_sala and res['fecha'] == fecha:
                ocupado = next((k for k, v in TURNOS.items() if v == res['turno']), None)
                if ocupado in turnos_libres:
                    turnos_libres.remove(ocupado)
        if turnos_libres:
            disponibles[clave_sala] = {
                'nombre': s['nombre'],
                'cupo': s['cupo'],
                'turnos_disponibles': sorted(turnos_libres)
            }
    return disponibles

def registrar_reserva():
    """Registra una nueva reservación, validando cliente, sala, fecha y turno disponible."""
    print("\n--- Registrar reservación ---")
    if not clientes or not salas:
        print("Debe haber al menos un cliente y una sala registrada.\n")
        return

    mostrar_clientes()
    while True:
        clave_cliente = input("Clave del cliente: ").strip()
        if clave_cliente.upper() == 'C':
            print("Operación cancelada.")
            return
        if clave_cliente not in clientes:
            print("Clave no válida.")
            continue
        break

    while True:
        fecha_txt = input("Fecha (DD/MM/YYYY): ").strip()
        if fecha_txt.upper() == 'C':
            print("Operación cancelada.")
            return
        fecha = convertir_fecha(fecha_txt)
        if not fecha:
            print("Formato de fecha inválido.")
            continue
        if not fecha_valida(fecha):
            print("Debe reservar con al menos 2 días de anticipación.")
            continue
        break

    disponibles = salas_disponibles(fecha)
    if not disponibles:
        print("No hay salas disponibles para esa fecha.\n")
        return

    print(f"\nSalas disponibles para {fecha.strftime('%d/%m/%Y')}:")
    print(f"{'Clave':6} | {'Sala':20} | {'Turnos disponibles':30} | {'Cupo':4}")
    print("-" * 80)
    for clave_sala, info in disponibles.items():
        turnos_txt = ", ".join([f"{k}-{TURNOS[k]}" for k in info['turnos_disponibles']])
        print(f"{clave_sala:6} | {info['nombre'][:20]:20} | {turnos_txt:30} | {str(info['cupo']):4}")

    while True:
        sala_sel = input("Clave de sala: ").strip()
        if sala_sel.upper() == 'C':
            print("Operación cancelada.")
            return
        if sala_sel not in disponibles:
            print("Sala no válida.")
            continue
        break

    posibles_turnos = disponibles[sala_sel]['turnos_disponibles']
    turnos_mostrables = [t for t in TURNOS.keys() if t in posibles_turnos]
    turnos_texto = "/".join(turnos_mostrables)
    print(f"Turnos disponibles para esta sala: ({turnos_texto})")

    while True:
        turno_sel = input(f"Selecciona el turno ({turnos_texto}): ").strip()
        if turno_sel.upper() == 'C':
            print("Operación cancelada.")
            return
        if turno_sel not in posibles_turnos:
            print("Turno no disponible, elige uno válido.")
            continue
        break

    while True:
        evento = input("Nombre del evento: ").strip()
        if evento.upper() == 'C':
            print("Operación cancelada.")
            return
        if not es_nombre_valido(evento):
            print("El nombre del evento solo puede contener letras y espacios.")
            continue
        break

    folio = siguiente_folio_reserva()
    reservas.append({
        'folio': folio,
        'cliente_id': clave_cliente,
        'sala_id': sala_sel,
        'fecha': fecha,
        'turno': TURNOS[turno_sel],
        'evento': evento
    })
    print(f"\nReservación registrada con folio: {folio}\n")

def consultar_y_exportar_reservas():
    """Permite consultar las reservaciones de una fecha específica y exportarlas a JSON."""
    if not reservas:
        print("\nNo hay reservaciones.\n")
        return

    fecha_txt = input("Fecha a consultar (DD/MM/YYYY): ").strip()
    fecha = convertir_fecha(fecha_txt)
    if not fecha:
        print("Formato de fecha inválido.\n")
        return

    filtradas = [r for r in reservas if r['fecha'] == fecha]
    if not filtradas:
        print("No hay reservaciones para esa fecha.\n")
        return

    print(f"\nReservaciones del {fecha.strftime('%d/%m/%Y')}:")
    print(f"{'Folio':8} | {'Turno':10} | {'Sala':15} | {'Cliente':20} | {'Evento':30}")
    print("-" * 90)
    for r in filtradas:
        sala = salas[r['sala_id']]['nombre']
        cliente = f"{clientes[r['cliente_id']]['apellidos']}, {clientes[r['cliente_id']]['nombre']}"
        print(f"{r['folio']:8} | {r['turno']:10} | {sala[:15]:15} | {cliente[:20]:20} | {r['evento'][:30]:30}")
    print()

    while True:
        resp = input("¿Deseas exportar estas reservaciones a JSON? (S/N): ").strip().upper()
        if resp in ('S','N'):
            break
    if resp == 'S':
        exportar_reservas_json(fecha, filtradas)

def exportar_reservas_json(fecha, lista):
    """Exporta una lista de reservaciones a un archivo JSON con nombre basado en la fecha."""
    nombre_archivo = f"reservas_{fecha.strftime('%d%m%Y')}.json"
    data = []
    for r in lista:
        data.append({
            'folio': r['folio'],
            'fecha': r['fecha'].strftime('%d/%m/%Y'),
            'turno': r['turno'],
            'sala': salas[r['sala_id']]['nombre'],
            'cliente': f"{clientes[r['cliente_id']]['apellidos']}, {clientes[r['cliente_id']]['nombre']}",
            'evento': r['evento']
        })
    with open(nombre_archivo, 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
    print(f"Reporte exportado correctamente como {nombre_archivo}\n")

def editar_evento():
    """Permite modificar el nombre de un evento existente usando su folio."""
    if not reservas:
        print("\nNo hay reservaciones.\n")
        return

    print("\n--- Editar nombre de evento ---")
    while True:
        inicio_txt = input("Fecha inicio (DD/MM/YYYY): ").strip()
        if inicio_txt.upper() == 'C':
            print("Operación cancelada.")
            return
        inicio = convertir_fecha(inicio_txt)
        if inicio:
            break
        print("Formato inválido.")

    while True:
        fin_txt = input("Fecha fin (DD/MM/YYYY): ").strip()
        if fin_txt.upper() == 'C':
            print("Operación cancelada.")
            return
        fin = convertir_fecha(fin_txt)
        if fin and fin >= inicio:
            break
        print("Fecha inválida o anterior a la inicial.")

    filtradas = [r for r in reservas if inicio <= r['fecha'] <= fin]
    if not filtradas:
        print("No hay reservaciones en ese rango.\n")
        return

    print(f"\nReservaciones del {inicio.strftime('%d/%m/%Y')} al {fin.strftime('%d/%m/%Y')}:")
    print(f"{'Folio':8} | {'Fecha':10} | {'Sala':15} | {'Evento':30}")
    print("-" * 70)
    for r in filtradas:
        print(f"{r['folio']:8} | {r['fecha'].strftime('%d/%m/%Y'):10} | {salas[r['sala_id']]['nombre'][:15]:15} | {r['evento'][:30]:30}")
    print()

    while True:
        folio_sel = input("Folio del evento a editar: ").strip()
        if folio_sel.upper() == 'C':
            print("Operación cancelada.")
            return
        reserva = next((r for r in reservas if r['folio'] == folio_sel), None)
        if reserva:
            break
        print("Folio no válido.")

    while True:
        nuevo_evento = input("Nuevo nombre del evento: ").strip()
        if nuevo_evento.upper() == 'C':
            print("Operación cancelada.")
            return
        if es_nombre_valido(nuevo_evento):
            break
        print("El nombre solo puede contener letras y espacios.")

    reserva['evento'] = nuevo_evento
    print("Nombre del evento actualizado correctamente.\n")

def confirmar_y_salir():
    """Pregunta al usuario si desea guardar los cambios antes de salir del sistema."""
    while True:
        resp = input("¿Deseas guardar los cambios y salir? (S/N): ").strip().upper()
        if resp in ('S','N'):
            break
    if resp == 'S':
        guardar_estado()
        print("Saliendo... ¡Hasta luego!")
        exit()
    else:
        print("Regresando al menú principal.\n")

def menu_principal():
    """Muestra el menú principal del sistema y gestiona la interacción del usuario."""
    opciones = {
        '1': ('Registrar/Mostrar clientes', lambda: (registrar_cliente(), mostrar_clientes())),
        '2': ('Registrar/Mostrar salas', lambda: (registrar_sala(), mostrar_salas())),
        '3': ('Registrar reservación', registrar_reserva),
        '4': ('Consultar y exportar reservaciones (JSON)', consultar_y_exportar_reservas),
        '5': ('Editar nombre de evento', editar_evento),
        '6': ('Salir', confirmar_y_salir)
    }

    while True:
        print("\n===== MENÚ PRINCIPAL =====")
        for k, (desc, _) in opciones.items():
            print(f"{k}. {desc}")
        op = input("Selecciona una opción: ").strip()
        if op in opciones:
            accion = opciones[op][1]
            accion()
        else:
            print("Opción inválida.\n")


if __name__ == "__main__":
    print("Sistema de Reservación de Salas - Coworking")
    print("Escribe 'C' para cancelar en cualquier momento.\n")
    cargar_estado()
    menu_principal()
