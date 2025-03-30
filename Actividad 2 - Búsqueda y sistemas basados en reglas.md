from experta import *

# DEFINICIÓN DE HECHOS (BASE DE CONOCIMIENTO)

class Ruta(Fact):
    """Representa una ruta del sistema con sus estaciones en orden"""
    pass

class Transferencia(Fact):
    """Indica estaciones donde se puede cambiar de ruta"""
    pass

class Recorrido(Fact):
    """Almacena una posible ruta sugerida con sus características"""
    pass

# MOTOR DE INFERENCIA (SISTEMA EXPERTO)

class PlanificadorRutas(KnowledgeEngine):
    @DefFacts()
    def cargar_datos(self):
        """Carga la base de conocimiento con rutas y estaciones de transferencia"""
        
        # Rutas principales del sistema (datos de ejemplo)
        yield Ruta(nombre="H72", estaciones=['Portal 80', 'Calle 76', 'Calle 72', 'Marly'])
        yield Ruta(nombre="G12", estaciones=['Marly', 'Calle 45', 'Calle 57', 'Portal Sur'])
        yield Ruta(nombre="F14", estaciones=['Portal Americas', 'Marly', 'Calle 22', 'Jiménez'])
        
        # Estaciones de transferencia entre rutas
        yield Transferencia(estacion="Marly", rutas=["H72", "G12", "F14"])
        yield Transferencia(estacion="Calle 72", rutas=["H72", "G12"])

    def __init__(self):
        super().__init__()
        self.origen = None
        self.destino = None
        self.mejor_ruta = None

    # REGLAS DE INFERENCIA

    @Rule(
        Fact(origen=MATCH.ori),
        Fact(destino=MATCH.des),
        Ruta(nombre=MATCH.ruta, estaciones=MATCH.estaciones),
        TEST(lambda estaciones, ori, des: ori in estaciones and des in estaciones),
        TEST(lambda estaciones, ori, des: estaciones.index(ori) < estaciones.index(des))
    )
    def ruta_directa(self, ruta, estaciones, ori, des):
        """Regla para rutas sin transbordos (misma línea)"""
        
        # Obtiene segmento de estaciones entre origen y destino
        idx_ini = estaciones.index(ori)
        idx_fin = estaciones.index(des)
        trayecto = estaciones[idx_ini:idx_fin+1]
        
        # Registra el recorrido encontrado
        self.declare(Recorrido(
            tipo='Directa',
            ruta=ruta,
            trayecto=trayecto,
            transbordos=0,
            estaciones=len(trayecto)
        ))

    @Rule(
        Fact(origen=MATCH.ori),
        Fact(destino=MATCH.des),
        Ruta(nombre=MATCH.r1, estaciones=MATCH.est1),
        Ruta(nombre=MATCH.r2, estaciones=MATCH.est2),
        Transferencia(estacion=MATCH.trans, rutas=MATCH.rutas),
        TEST(lambda ori, est1: ori in est1),
        TEST(lambda des, est2: des in est2),
        TEST(lambda r1, r2, rutas: r1 in rutas and r2 in rutas and r1 != r2),
        TEST(lambda trans, est1, est2: trans in est1 and trans in est2),
        TEST(lambda ori, trans, est1: est1.index(ori) < est1.index(trans)),
        TEST(lambda des, trans, est2: est2.index(trans) < est2.index(des))
    )
    def ruta_con_transbordo(self, r1, r2, trans, est1, est2, ori, des):
        """Regla para rutas que requieren cambio de línea"""
        
        # Segmento desde origen hasta punto de transferencia
        idx_ini = est1.index(ori)
        idx_trans = est1.index(trans)
        tramo1 = est1[idx_ini:idx_trans+1]
        
        # Segmento desde transferencia hasta destino
        idx_trans = est2.index(trans)
        idx_fin = est2.index(des)
        tramo2 = est2[idx_trans:idx_fin+1]
        
        # Combina ambos tramos (evitando duplicar estación de transferencia)
        ruta_completa = tramo1 + tramo2[1:]
        
        # Registra el recorrido encontrado
        self.declare(Recorrido(
            tipo='Con transbordo',
            ruta=f"{r1} → {r2}",
            trayecto=ruta_completa,
            transbordos=1,
            estaciones=len(ruta_completa)
        ))

    # FUNCIONES AUXILIARES

    def seleccionar_mejor_ruta(self):
        """Evalúa y selecciona la mejor ruta según criterios"""
        opciones = [h for h in self.facts.values() if isinstance(h, Recorrido)]
        
        if not opciones:
            return None
        
        # Criterio: menos transbordos > menos estaciones
        return min(opciones, key=lambda x: (x['transbordos'], x['estaciones']))

    def planificar(self, origen, destino):
        """Interfaz principal para ejecutar la planificación"""
        self.reset()
        self.origen = origen
        self.destino = destino
        
        # Declara hechos iniciales
        self.declare(Fact(origen=origen))
        self.declare(Fact(destino=destino))
        
        # Ejecuta el motor de inferencia
        self.run()
        
        # Obtiene el mejor recorrido
        self.mejor_ruta = self.seleccionar_mejor_ruta()
        return self.mejor_ruta

# INTERFAZ DE USUARIO

def mostrar_menu_estaciones():
    """Muestra estaciones disponibles para selección"""
    print("\nEstaciones disponibles:")
    print("1. Portal 80")
    print("2. Calle 76")
    print("3. Calle 72")
    print("4. Marly")
    print("5. Calle 45")
    print("6. Calle 57")
    print("7. Portal Sur")
    print("8. Portal Americas")
    print("9. Calle 22")
    print("10. Jiménez")

def obtener_estacion(opcion):
    """Convierte selección numérica en nombre de estación"""
    estaciones = {
        1: "Portal 80",
        2: "Calle 76",
        3: "Calle 72",
        4: "Marly",
        5: "Calle 45",
        6: "Calle 57",
        7: "Portal Sur",
        8: "Portal Americas",
        9: "Calle 22",
        10: "Jiménez"
    }
    return estaciones.get(opcion, None)

def main():
    """Función principal del programa"""
    print("SISTEMA DE PLANIFICACIÓN DE RUTAS - TRANSMILENIO")
    
    # Mostrar estaciones disponibles
    mostrar_menu_estaciones()
    
    # Solicitar origen y destino
    try:
        origen = obtener_estacion(int(input("\nIngrese número de estación ORIGEN: ")))
        destino = obtener_estacion(int(input("Ingrese número de estación DESTINO: ")))
        
        if not origen or not destino:
            print("Error: Selección inválida")
            return
            
        if origen == destino:
            print("El origen y destino no pueden ser iguales")
            return
            
    except ValueError:
        print("Error: Debe ingresar un número válido")
        return
    
    # Ejecutar planificación
    sistema = PlanificadorRutas()
    resultado = sistema.planificar(origen, destino)
    
    # Mostrar resultados
    print("\n" + "="*50)
    if resultado:
        print(f"\nMEJOR RUTA ENCONTRADA ({resultado['tipo']}):")
        print(" → ".join(resultado['trayecto']))
        print(f"\nDetalles:")
        print(f"- Línea(s): {resultado['ruta']}")
        print(f"- Transbordos: {resultado['transbordos']}")
        print(f"- Estaciones totales: {resultado['estaciones']}")
    else:
        print("\nNo se encontró una ruta válida entre las estaciones seleccionadas")
    print("="*50)

if __name__ == "__main__":
    main()
