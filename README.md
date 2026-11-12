"""
Software FJ - Sistema Integral de Gestión de Clientes, Servicios y Reservas
UNAD - Programación 213023 - ECBTI
Sin base de datos. Solo listas + archivo de logs.
"""
from abc import ABC, abstractmethod
from datetime import datetime
import traceback

# ============ LOGS ============
LOG_FILE = "logs_sistema.txt"
def registrar_log(nivel, mensaje):
    linea = f"[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] [{nivel}] {mensaje}\n"
    try:
        with open(LOG_FILE, "a", encoding="utf-8") as f:
            f.write(linea)
    except Exception:
        print("No se pudo escribir log:", linea)

# ============ EXCEPCIONES PERSONALIZADAS ============
class ErrorSistema(Exception): pass
class DatosInvalidosError(ErrorSistema): pass
class ClienteInvalidoError(DatosInvalidosError): pass
class ServicioNoDisponibleError(ErrorSistema): pass
class ReservaInvalidaError(ErrorSistema): pass
class OperacionNoPermitidaError(ErrorSistema): pass

# ============ CLASE ABSTRACTA GENERAL ============
class Entidad(ABC):
    def __init__(self, id_entidad):
        if not id_entidad:
            raise DatosInvalidosError("El ID es obligatorio")
        self._id = id_entidad
        self._fecha_creacion = datetime.now()

    @property
    def id(self): return self._id

    @abstractmethod
    def describir(self): pass
    @abstractmethod
    def validar(self): pass

# ============ CLIENTE ============
class Cliente(Entidad):
    def __init__(self, id_cliente, nombre, correo, telefono, edad):
        super().__init__(id_cliente)
        self.__nombre = nombre
        self.__correo = correo
        self.__telefono = telefono
        self.__edad = edad
        self.validar()

    @property
    def nombre(self): return self.__nombre
    @nombre.setter
    def nombre(self, v):
        if not v or len(v.strip()) < 3:
            raise ClienteInvalidoError("Nombre debe tener mínimo 3 caracteres")
        self.__nombre = v.strip()

    @property
    def correo(self): return self.__correo
    @correo.setter
    def correo(self, v):
        if "@" not in v or "." not in v:
            raise ClienteInvalidoError(f"Correo inválido: {v}")
        self.__correo = v

    @property
    def edad(self): return self.__edad
    @edad.setter
    def edad(self, v):
        if not isinstance(v, int) or v < 18:
            raise ClienteInvalidoError("Cliente debe ser mayor de 18 años")
        self.__edad = v

    @property
    def telefono(self): return self.__telefono
    @telefono.setter
    def telefono(self, v):
        if not str(v).isdigit() or len(str(v)) < 7:
            raise ClienteInvalidoError("Teléfono inválido")
        self.__telefono = str(v)

    def validar(self):
        self.nombre = self.__nombre
        self.correo = self.__correo
        self.telefono = self.__telefono
        self.edad = self.__edad

    def describir(self):
        return f"Cliente [{self.id}] {self.__nombre} - {self.__correo} - Edad: {self.__edad}"

# ============ SERVICIO ABSTRACTO ============
class Servicio(Entidad):
    def __init__(self, id_servicio, nombre, precio_base, disponible=True):
        super().__init__(id_servicio)
        self._nombre = nombre
        self._precio_base = precio_base
        self._disponible = disponible
        self.validar()

    @property
    def nombre(self): return self._nombre
    @property
    def precio_base(self): return self._precio_base
    @property
    def disponible(self): return self._disponible
    @disponible.setter
    def disponible(self, v): self._disponible = bool(v)

    def validar(self):
        if not self._nombre:
            raise DatosInvalidosError("Servicio sin nombre")
        if not isinstance(self._precio_base, (int, float)) or self._precio_base <= 0:
            raise DatosInvalidosError(f"Precio base inválido: {self._precio_base}")

    # Métodos sobrecargados (variantes con parámetros opcionales)
    def calcular_costo(self, duracion=1, impuesto=0.0, descuento=0.0):
        raise NotImplementedError("Implementar en clase hija")

    def calcular_costo_con_impuesto(self, duracion=1):
        return self.calcular_costo(duracion, impuesto=0.19)

    def calcular_costo_con_descuento(self, duracion=1, descuento=0.10):
        return self.calcular_costo(duracion, descuento=descuento)

    @abstractmethod
    def describir(self): pass

class ReservaSala(Servicio):
    def __init__(self, id_servicio, nombre, precio_base, capacidad, disponible=True):
        self.capacidad = capacidad
        super().__init__(id_servicio, nombre, precio_base, disponible)

    def validar(self):
        super().validar()
        if not isinstance(self.capacidad, int) or self.capacidad <= 0:
            raise DatosInvalidosError("Capacidad de sala inválida")

    def calcular_costo(self, duracion=1, impuesto=0.0, descuento=0.0):
        if duracion <= 0: raise DatosInvalidosError("Duración debe ser > 0")
        return round(self._precio_base * duracion * (1 - descuento) * (1 + impuesto), 2)

    def describir(self):
        return f"Sala [{self.id}] {self._nombre} - Cap:{self.capacidad} - ${self._precio_base}/hora - Disponible:{self._disponible}"

class AlquilerEquipos(Servicio):
    def __init__(self, id_servicio, nombre, precio_base, cantidad, disponible=True):
        self.cantidad = cantidad
        super().__init__(id_servicio, nombre, precio_base, disponible)

    def validar(self):
        super().validar()
        if self.cantidad <= 0: raise DatosInvalidosError("Cantidad de equipos inválida")

    def calcular_costo(self, duracion=1, impuesto=0.0, descuento=0.0):
        if duracion <= 0: raise DatosInvalidosError("Duración debe ser > 0")
        return round(self._precio_base * duracion * self.cantidad * (1-descuento)*(1+impuesto), 2)

    def describir(self):
        return f"Equipo [{self.id}] {self._nombre} x{self.cantidad} - ${self._precio_base}/día - Disponible:{self._disponible}"

class AsesoriaEspecializada(Servicio):
    def __init__(self, id_servicio, nombre, precio_base, nivel="Junior", disponible=True):
        self.nivel = nivel
        super().__init__(id_servicio, nombre, precio_base, disponible)

    def validar(self):
        super().validar()
        if self.nivel not in ["Junior", "Senior", "Experto"]:
            raise DatosInvalidosError("Nivel debe ser Junior, Senior o Experto")

    def calcular_costo(self, duracion=1, impuesto=0.0, descuento=0.0):
        if duracion <= 0: raise DatosInvalidosError("Duración debe ser > 0")
        factor = {"Junior":1.0, "Senior":1.5, "Experto":2.0}[self.nivel]
        return round(self._precio_base * duracion * factor * (1-descuento)*(1+impuesto), 2)

    def describir(self):
        return f"Asesoría [{self.id}] {self._nombre} ({self.nivel}) - ${self._precio_base}/hora - Disponible:{self._disponible}"

# ============ RESERVA ============
class Reserva(Entidad):
    def __init__(self, id_reserva, cliente, servicio, duracion):
        super().__init__(id_reserva)
        self.cliente = cliente
        self.servicio = servicio
        self.duracion = duracion
        self.estado = "Pendiente"
        self.validar()

    def validar(self):
        if not isinstance(self.cliente, Cliente):
            raise ReservaInvalidaError("Cliente inválido para reserva")
        if not isinstance(self.servicio, Servicio):
            raise ReservaInvalidaError("Servicio inválido para reserva")
        if not isinstance(self.duracion, (int, float)) or self.duracion <= 0:
            raise ReservaInvalidaError("Duración inválida")
        if not self.servicio.disponible:
            raise ServicioNoDisponibleError(f"Servicio {self.servicio.nombre} no disponible")

    def confirmar(self):
        try:
            if self.estado != "Pendiente":
                raise OperacionNoPermitidaError(f"No se puede confirmar reserva en estado {self.estado}")
            self.estado = "Confirmada"
            registrar_log("INFO", f"Reserva {self.id} confirmada - Costo: ${self.servicio.calcular_costo(self.duracion)}")
            return True
        except Exception as e:
            registrar_log("ERROR", f"Error confirmar {self.id}: {e}")
            raise

    def cancelar(self):
        try:
            if self.estado == "Cancelada":
                raise OperacionNoPermitidaError("Reserva ya cancelada")
            if self.estado == "Procesada":
                raise OperacionNoPermitidaError("No se puede cancelar reserva procesada")
            self.estado = "Cancelada"
            registrar_log("INFO", f"Reserva {self.id} cancelada")
        except Exception as e:
            registrar_log("ERROR", f"Error cancelar {self.id}: {e}")
            raise

    def procesar(self):
        try:
            if self.estado != "Confirmada":
                raise OperacionNoPermitidaError("Solo se puede procesar reserva Confirmada")
            costo = self.servicio.calcular_costo(self.duracion, impuesto=0.19, descuento=0.05)
            self.estado = "Procesada"
            registrar_log("INFO", f"Reserva {self.id} procesada - Costo final: ${costo}")
            return costo
        except Exception as e:
            # Encadenamiento de excepciones
            raise OperacionNoPermitidaError(f"Fallo al procesar reserva {self.id}") from e

    def describir(self):
        return f"Reserva [{self.id}] {self.cliente.nombre} -> {self.servicio.nombre} ({self.duracion}h) - {self.estado}"

# ============ SISTEMA ============
class SistemaFJ:
    def __init__(self):
        self.clientes = []
        self.servicios = []
        self.reservas = []

    def agregar_cliente(self, cliente):
        self.clientes.append(cliente)
        registrar_log("INFO", f"Cliente agregado: {cliente.describir()}")
        print("✅", cliente.describir())

    def agregar_servicio(self, servicio):
        self.servicios.append(servicio)
        print("✅", servicio.describir())

    def crear_reserva(self, id_reserva, cliente, servicio, duracion):
        reserva = Reserva(id_reserva, cliente, servicio, duracion)
        self.reservas.append(reserva)
        print("✅", reserva.describir())
        return reserva

    def listar_todo(self):
        print("\n--- CLIENTES ---")
        for c in self.clientes: print(c.describir())
        print("\n--- SERVICIOS (Polimorfismo) ---")
        for s in self.servicios:
            print(s.describir(), "| Costo 2h:", s.calcular_costo(2), "| Con impuesto:", s.calcular_costo_con_impuesto(2), "| Con descuento:", s.calcular_costo_con_descuento(2))
        print("\n--- RESERVAS ---")
        for r in self.reservas: print(r.describir())

# ============ SIMULACIÓN 12 OPERACIONES ============
def main():
    sistema = SistemaFJ()
    print("=== SOFTWARE FJ - INICIO ===")
    registrar_log("INFO", "Inicio del sistema")

    # 1 Cliente válido - try/except/else/finally
    try:
        c1 = Cliente("C001", "Felipe Jaramillo", "felipe@mail.com", "3001234567", 25)
        sistema.agregar_cliente(c1)
    except Exception as e:
        print("❌ Op1:", e)
    else:
        print("Op1 OK - else ejecutado")
    finally:
        print("Op1 finally: continúa sistema")

    # 2 Cliente inválido
    try:
        c2 = Cliente("C002", "Ana", "ana-sin-correo", "123", 16)
        sistema.agregar_cliente(c2)
    except ClienteInvalidoError as e:
        print("❌ Op2 capturado:", e); registrar_log("ERROR", f"Op2: {e}")

    # 3 Cliente menor de edad
    try:
        c3 = Cliente("C003", "Juan Perez", "juan@mail.com", "3001112222", 15)
        sistema.agregar_cliente(c3)
    except Exception as e: print("❌ Op3:", e)

    # 4 Servicio válido Sala
    try:
        s1 = ReservaSala("S001", "Sala Reuniones A", 50000, 10)
        sistema.agregar_servicio(s1)
    except Exception as e: print("❌ Op4:", e)

    # 5 Servicio inválido - precio negativo
    try:
        s_bad = AlquilerEquipos("S002", "Proyector", -100, 2)
        sistema.agregar_servicio(s_bad)
    except DatosInvalidosError as e: print("❌ Op5:", e)

    # 6 Servicio válido Equipos
    try:
        s2 = AlquilerEquipos("S003", "Portátil HP", 40000, 5)
        sistema.agregar_servicio(s2)
    except Exception as e: print("❌ Op6:", e)

    # 7 Servicio válido Asesoría
    try:
        s3 = AsesoriaEspecializada("S004", "Asesoría Python", 80000, "Experto")
        sistema.agregar_servicio(s3)
    except Exception as e: print("❌ Op7:", e)

    # 8 Servicio no disponible
    try:
        s4 = ReservaSala("S005", "Sala B", 60000, 5, disponible=False)
        sistema.agregar_servicio(s4)
        sistema.crear_reserva("R001", c1, s4, 2)
    except ServicioNoDisponibleError as e: print("❌ Op8:", e)

    # 9 Reserva exitosa + confirmar + procesar
    try:
        r1 = sistema.crear_reserva("R002", c1, s1, 3)
        r1.confirmar()
        print("Costo procesado:", r1.procesar())
    except Exception as e: print("❌ Op9:", e)

    # 10 Reserva inválida duración 0
    try:
        sistema.crear_reserva("R003", c1, s2, 0)
    except ReservaInvalidaError as e: print("❌ Op10:", e)

    # 11 Reserva + cancelar + doble cancelación + procesar cancelada
    try:
        r3 = sistema.crear_reserva("R004", c1, s3, 2)
        r3.confirmar(); r3.cancelar()
        try: r3.cancelar()
        except Exception as e2: print("❌ Op11 doble cancelar:", e2)
        try: r3.procesar()
        except Exception as e3: print("❌ Op11 procesar cancelada:", e3, "| Causa:", e3.__cause__)
    except Exception as e: print("❌ Op11:", e)

    # 12 Cálculo inconsistente
    try:
        print("Costo con duración None:", s1.calcular_costo(None))
    except Exception as e:
        print("❌ Op12:", e); registrar_log("ERROR", f"Op12: {e}")

    sistema.listar_todo()
    print("\n=== FIN - Sistema sigue activo. Revisa logs_sistema.txt ===")

if __name__ == "__main__":
    try:
        main()
    except Exception as e:
        registrar_log("CRITICO", f"Fallo general: {e}")
        print("Error crítico controlado, sistema no se cayó:", e)
    finally:
        print("Finally global: limpieza y cierre de logs.")
        
