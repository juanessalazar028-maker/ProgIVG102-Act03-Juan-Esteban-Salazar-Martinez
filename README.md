import textwrap
from abc import ABC, abstractmethod
from datetime import date, timedelta

class Material(ABC):
    """Clase base abstracta: lo comun a todo material de la biblioteca."""

    def __init__(self, titulo, autor, isbn, ejemplares_totales, **detalles):
        self.titulo = titulo
        self.autor = autor
        self.isbn = isbn
        self.ejemplares_totales = ejemplares_totales
        self._ejemplares_disponibles = ejemplares_totales
        self.detalles = detalles  # datos opcionales: editorial, año, categoria...

    @property
    def disponibles(self):
        """Getter: cantidad actual de ejemplares (o licencias) disponibles."""
        return self._ejemplares_disponibles

    @disponibles.setter
    def disponibles(self, valor):
        """Setter: valida que el valor este entre 0 y ejemplares_totales."""
        if valor < 0 or valor > self.ejemplares_totales:
            raise ValueError("disponibles debe estar entre 0 y ejemplares_totales")
        self._ejemplares_disponibles = valor

    def esta_disponible(self):
        """True si queda al menos un ejemplar disponible."""
        return self.disponibles > 0

    def prestar(self):
        """Reduce en 1 los ejemplares disponibles de este material."""
        if not self.esta_disponible():
            raise ValueError(f"No hay ejemplares disponibles de {self.titulo}")
        self.disponibles = self.disponibles - 1

    def devolver(self):
        """Aumenta en 1 los ejemplares disponibles de este material."""
        self.disponibles = self.disponibles + 1

    @abstractmethod
    def dias_prestamo(self):
        """Cantidad de dias que puede prestarse este material."""

    def __str__(self):
        return f"{self.titulo} - {self.autor}"

    def __repr__(self):
        return (f"{type(self).__name__}(titulo={self.titulo!r}, "
                f"autor={self.autor!r}, isbn={self.isbn!r}, "
                f"ejemplares_totales={self.ejemplares_totales!r})")


class LibroFisico(Material):
    """Libro en papel: ademas de lo comun, tiene ubicacion en el estante."""

    DIAS_PRESTAMO = 14  # plazo del libro fisico, en un solo lugar

    def __init__(self, titulo, autor, isbn, ejemplares_totales,
                 ubicacion="Sin asignar", **detalles):
        super().__init__(titulo, autor, isbn, ejemplares_totales, **detalles)
        self.ubicacion = ubicacion

    def dias_prestamo(self):
        """Un libro fisico se presta por DIAS_PRESTAMO (14) dias."""
        return LibroFisico.DIAS_PRESTAMO

    def __str__(self):
        return f"{super().__str__()} [Fisico, estante {self.ubicacion}]"

    def __repr__(self):
        return (f"{type(self).__name__}(titulo={self.titulo!r}, "
                f"autor={self.autor!r}, isbn={self.isbn!r}, "
                f"ejemplares_totales={self.ejemplares_totales!r}, "
                f"ubicacion={self.ubicacion!r})")


class RecursoDigital(Material):
    """Recurso en linea: ademas de lo comun, tiene formato y enlace de acceso."""

    DIAS_PRESTAMO = 7  # plazo del recurso digital, en un solo lugar

    def __init__(self, titulo, autor, isbn, ejemplares_totales,
                 formato="PDF", url="", **detalles):
        super().__init__(titulo, autor, isbn, ejemplares_totales, **detalles)
        self.formato = formato
        self.url = url

    def dias_prestamo(self):
        """Un recurso digital se presta por DIAS_PRESTAMO (7) dias."""
        return RecursoDigital.DIAS_PRESTAMO

    def __str__(self):
        return f"{super().__str__()} [Digital, {self.formato}]"

    def __repr__(self):
        return (f"{type(self).__name__}(titulo={self.titulo!r}, "
                f"autor={self.autor!r}, isbn={self.isbn!r}, "
                f"ejemplares_totales={self.ejemplares_totales!r}, "
                f"formato={self.formato!r}, url={self.url!r})")


class LibroHibrido(LibroFisico, RecursoDigital):
    """Libro que existe en papel y en digital a la vez.
    """

    def dias_prestamo(self):
        """Se aplica el plazo mas corto de los dos padres.

        Se nombran las constantes de cada clase de forma explicita
        """
        return min(LibroFisico.DIAS_PRESTAMO, RecursoDigital.DIAS_PRESTAMO)

    def __repr__(self):
        return (f"{type(self).__name__}(titulo={self.titulo!r}, "
                f"autor={self.autor!r}, isbn={self.isbn!r}, "
                f"ejemplares_totales={self.ejemplares_totales!r}, "
                f"ubicacion={self.ubicacion!r}, formato={self.formato!r})")


class Usuario:
    """Representa a la persona que puede solicitar libros en prestamo."""

    def __init__(self, nombre, codigo_estudiantil, tipo_usuario="estudiante"):
        self.nombre = nombre
        self._codigo_estudiantil = codigo_estudiantil
        self.tipo_usuario = tipo_usuario

    @property
    def codigo_estudiantil(self):
        """Solo lectura: el codigo no deberia cambiar tras crear el usuario."""
        return self._codigo_estudiantil

    @staticmethod
    def es_codigo_valido(codigo):
        """Valida un codigo (10 digitos) sin necesitar un Usuario creado."""
        return codigo.isdigit() and len(codigo) == 10

    def __str__(self):
        return f"{self.nombre} ({self.tipo_usuario})"

    def __repr__(self):
        return (f"Usuario(nombre={self.nombre!r}, "
                f"codigo_estudiantil={self.codigo_estudiantil!r}, "
                f"tipo_usuario={self.tipo_usuario!r})")


class Prestamo:
    """Relaciona un material con un Usuario en un momento dado."""

    def __init__(self, material, usuario, fecha_inicio, fecha_devolucion_esperada):
        self.material = material
        self.usuario = usuario
        self.fecha_inicio = fecha_inicio
        self.fecha_devolucion_esperada = fecha_devolucion_esperada

    @property
    def libro(self):
        """Alias de compatibilidad: ComprobantePrestamo sigue leyendo .libro."""
        return self.material

    @classmethod
    def registrar_hoy(cls, material, usuario, dias_prestamo=None):
        """Registra el prestamo hoy; si no se fuerza un plazo, el material decide."""
        hoy = date.today()
        if dias_prestamo is None:
            dias_prestamo = material.dias_prestamo()
        return cls(material, usuario, hoy, hoy + timedelta(days=dias_prestamo))

    def __str__(self):
        return (f"{self.material.titulo} prestado a {self.usuario.nombre}: "
                f"del {self.fecha_inicio} al {self.fecha_devolucion_esperada}")

    def __repr__(self):
        return (f"Prestamo(material={self.material!r}, usuario={self.usuario!r}, "
                f"fecha_inicio={self.fecha_inicio!r}, "
                f"fecha_devolucion_esperada={self.fecha_devolucion_esperada!r})")


def registrar_prestamo(material: Material, usuario):
    """Presta cualquier Material y registra el prestamo (principio de Liskov)."""
    prestamo = Prestamo.registrar_hoy(material, usuario)  # paso que puede fallar
    material.prestar()  # solo se descuenta el ejemplar si lo anterior funciono
    return prestamo


class ComprobantePrestamo:
    """Comprobante inmutable: una vez emitido, no puede alterarse."""

    __slots__ = ("_libro_titulo", "_usuario_nombre",
                 "_fecha_inicio", "_fecha_devolucion_esperada")

    def __init__(self, prestamo):
        self._libro_titulo = prestamo.libro.titulo
        self._usuario_nombre = prestamo.usuario.nombre
        self._fecha_inicio = prestamo.fecha_inicio
        self._fecha_devolucion_esperada = prestamo.fecha_devolucion_esperada

    # Las siguientes propiedades son de solo lectura (sin @setter):
    @property
    def libro_titulo(self):
        return self._libro_titulo

    @property
    def usuario_nombre(self):
        return self._usuario_nombre

    @property
    def fecha_inicio(self):
        return self._fecha_inicio

    @property
    def fecha_devolucion_esperada(self):
        return self._fecha_devolucion_esperada


if __name__ == "__main__":
    
    fisico = LibroFisico("Clean Code", "Robert C. Martin",
                         "978-0132350884", 2, ubicacion="B-12")
    digital = RecursoDigital("Fluent Python", "Luciano Ramalho",
                             "978-1492056355", 5, formato="EPUB")
    hibrido = LibroHibrido("Python 3 OOP", "Dusty Phillips",
                           "978-1789615852", 3, ubicacion="C-03", formato="PDF")
    usuario_1 = Usuario("Juan", "1093812345")

    def seccion(titulo):
        """Imprime un encabezado que separa cada parte de la salida."""
        print(f"\n=== {titulo} ===")

    def mostrar(texto, sangria="  "):
        """Imprime un texto largo partido en lineas de maximo 70 caracteres."""
        print(textwrap.fill(texto, width=70, initial_indent=sangria,
                            subsequent_indent=sangria + "  ",
                            break_long_words=False, break_on_hyphens=False))

    # Material es abstracta, no se puede instanciar
    seccion("Parte B: Material es abstracta")
    try:
        Material("Cualquiera", "Alguien", "123", 1)
    except TypeError:
        print("  No se puede crear un Material sin definir dias_prestamo()")

    # subclases (prestar y disponibles heredados, dias_prestamo propio)
    seccion("Parte C: subclases")
    fisico.prestar()
    for material in (fisico, digital):
        print(f"{material.titulo}")
        print(f"  quedan {material.disponibles} de "
              f"{material.ejemplares_totales} ejemplares")
        print(f"  se presta {material.dias_prestamo()} dias")
    print(f"LibroFisico es Material: {isinstance(fisico, Material)}")
    print(f"LibroFisico es RecursoDigital: "
          f"{isinstance(fisico, RecursoDigital)}")

    #  __str__ (para personas) y __repr__ (para programadores)
    seccion("Parte D: __str__ y __repr__")
    print("str(fisico):")
    mostrar(str(fisico))
    print("repr(fisico):")
    mostrar(repr(fisico))
    print("Lista [fisico, digital] (usa __repr__ de cada elemento):")
    for elemento in (fisico, digital):
        mostrar(repr(elemento))
    print("Usuario:")
    mostrar(str(usuario_1))

    # herencia multiple (orden de resolucion de metodos)
    seccion("Parte E: herencia multiple")
    print(f"{hibrido.titulo}")
    mostrar(str(hibrido))
    print(f"  se presta {hibrido.dias_prestamo()} dias")
    print("MRO (orden en que Python busca los metodos):")
    for orden, clase in enumerate(LibroHibrido.__mro__, start=1):
        print(f"  {orden}. {clase.__name__}")

    # el mismo ciclo trata distinto a cada tipo de material
    seccion("Parte F: prestamos")
    catalogo = [fisico, digital, hibrido]
    for material in catalogo:
        prestamo = registrar_prestamo(material, usuario_1)
        print(prestamo)

    comprobante = ComprobantePrestamo(prestamo)
    print(f"\nComprobante emitido para: {comprobante.libro_titulo}")
