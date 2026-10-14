import json
import shutil
import subprocess
import urllib.request
import urllib.error
from pathlib import Path

OUT = Path("resultados")
OUT.mkdir(exist_ok=True)


def guardar(nombre, texto):
    (OUT / nombre).write_text(texto, encoding="utf-8")


# ---------------- DIG ----------------

def dns_dig(dominio):
    print("\n[+] DIG: consulta de registros DNS")

    if not shutil.which("dig"):
        print("[!] No está instalado dig. Ejecuta: sudo apt install dnsutils")
        return

    resultados = []

    for tipo in ("A", "AAAA", "MX", "NS", "TXT"):
        print(f"\n--- Registro {tipo} ---")

        try:
            r = subprocess.run(
                ["dig", dominio, tipo, "+time=3", "+tries=1"],
                capture_output=True,
                text=True,
                timeout=15
            )

            salida = r.stdout or r.stderr or "Sin resultados"
            print(salida[:3000])
            resultados.append(f"--- {tipo} ---\n{salida}")

        except (subprocess.TimeoutExpired, OSError) as e:
            mensaje = f"Error consultando {tipo}: {e}"
            print("[!] " + mensaje)
            resultados.append(mensaje)

    guardar("dig.txt", "\n".join(resultados))
    print("\n[+] Guardado en resultados/dig.txt")


# ---------------- NSLOOKUP ----------------

def dns_nslookup(dominio):
    print("\n[+] NSLOOKUP: resolución DNS")

    if not shutil.which("nslookup"):
        print("[!] No está instalado nslookup. Ejecuta: sudo apt install dnsutils")
        return

    try:
        r = subprocess.run(
            ["nslookup", dominio],
            capture_output=True,
            text=True,
            timeout=15
        )

        salida = r.stdout or r.stderr or "Sin resultados"
        print(salida[:5000])
        guardar("nslookup.txt", salida)
        print("[+] Guardado en resultados/nslookup.txt")

    except (subprocess.TimeoutExpired, OSError) as e:
        print(f"[!] Error en NSLOOKUP: {e}")


# ---------------- FINGERPRINTING HTTP ----------------

def fingerprint_http(dominio):
    print("\n[+] FINGERPRINTING HTTP")

    datos = None

    for esquema in ("https", "http"):
        url = f"{esquema}://{dominio}/"

        peticion = urllib.request.Request(
            url,
            headers={"User-Agent": "BUSC4/2.0"}
        )

        try:
            with urllib.request.urlopen(peticion, timeout=10) as respuesta:
                datos = {
                    "url": url,
                    "status": respuesta.status,
                    "headers": dict(respuesta.headers.items())
                }
            break

        except urllib.error.HTTPError as e:
            # Las respuestas HTTP 403 o 404 también contienen información.
            datos = {
                "url": url,
                "status": e.code,
                "headers": dict(e.headers.items())
            }
            break

        except Exception as e:
            print(f"[-] {url}: {e}")

    if datos is None:
        print("[!] No se pudo obtener una respuesta HTTP.")
        return

    print(f"\nURL: {datos['url']}")
    print(f"Estado HTTP: {datos['status']}")

    for nombre, valor in datos["headers"].items():
        print(f"{nombre}: {valor}")

    guardar(
        "fingerprinting.json",
        json.dumps(datos, indent=2, ensure_ascii=False)
    )

    print("\n[+] Guardado en resultados/fingerprinting.json")


# ---------------- ENUMERACIÓN COMPLETA ----------------

def enumeracion_completa(dominio):
    print("\n" + "=" * 48)
    print(" BUSC4 - ENUMERACIÓN ACTIVA COMPLETA")
    print("=" * 48)

    print("\n[1/3] Consultas DIG")
    dns_dig(dominio)

    print("\n[2/3] Consultas NSLOOKUP")
    dns_nslookup(dominio)

    print("\n[3/3] Fingerprinting HTTP")
    fingerprint_http(dominio)

    print("\n[+] Enumeración activa completa finalizada.")


# ---------------- MENÚ ACTIVO ----------------

def menu_activa(dominio):
    while True:
        print("""
╔══════════════════════════════════════╗
║       BUSC4 - ENUMERACIÓN ACTIVA     ║
╠══════════════════════════════════════╣
║  [1] DIG                             ║
║  [2] NSLOOKUP                        ║
║  [3] Fingerprinting HTTP             ║
║  [4] ENUMERACIÓN ACTIVA COMPLETA     ║
║  [0] Volver al menú principal        ║
╚══════════════════════════════════════╝
""")

        opcion = input("BUSC4/ACTIVA > ").strip()

        if opcion == "0":
            break

        if opcion not in ("1", "2", "3", "4"):
            print("[!] Opción no válida.")
            continue

        print("\n[!] Las consultas DNS y HTTP generan tráfico.")
        confirmacion = input(
            "¿Tienes autorización para analizar este objetivo? (s/N): "
        ).strip().lower()

        if confirmacion != "s":
            print("[-] Operación cancelada.")
            continue

        if opcion == "1":
            dns_dig(dominio)

        elif opcion == "2":
            dns_nslookup(dominio)

        elif opcion == "3":
            fingerprint_http(dominio)

        elif opcion == "4":
            enumeracion_completa(dominio)

        input("\nPulsa ENTER para volver al menú activo...")