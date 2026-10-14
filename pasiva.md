import shutil
import subprocess
import urllib.parse
import urllib.request
from pathlib import Path

OUT = Path("resultados")
OUT.mkdir(exist_ok=True)


def guardar(nombre, texto):
    (OUT / nombre).write_text(texto, encoding="utf-8")


# ---------------- WHOIS ----------------

def whois(dominio):
    print("\n[+] Consultando WHOIS...")

    if not shutil.which("whois"):
        print("[!] Instala WHOIS: sudo apt install whois")
        return

    try:
        r = subprocess.run(
            ["whois", dominio],
            capture_output=True,
            text=True,
            timeout=25
        )
        texto = r.stdout or r.stderr or "Sin resultados"
        print(texto[:7000])
        guardar("whois.txt", texto)
        print("[+] Guardado en resultados/whois.txt")

    except (subprocess.TimeoutExpired, OSError) as e:
        print(f"[!] Error WHOIS: {e}")


# ---------------- WAYBACK MACHINE ----------------

def wayback(dominio):
    print("\n[+] Consultando Wayback Machine...")

    params = urllib.parse.urlencode({
        "url": dominio + "/*",
        "output": "json",
        "filter": "statuscode:200",
        "collapse": "urlkey",
        "fl": "timestamp,original,statuscode"
    })

    url = "https://web.archive.org/cdx/search/cdx?" + params

    try:
        req = urllib.request.Request(
            url,
            headers={"User-Agent": "BUSC4/2.0"}
        )

        with urllib.request.urlopen(req, timeout=20) as r:
            texto = r.read().decode("utf-8", errors="replace")

        guardar("wayback.txt", texto)

        filas = texto.splitlines()

        if len(filas) > 1:
            print("\nCapturas encontradas:")
            for fila in filas[1:16]:
                print("  " + fila)
            print(f"\n[+] Total de filas: {len(filas) - 1}")
        else:
            print("[i] No se encontraron capturas.")

        print("[+] Guardado en resultados/wayback.txt")

    except Exception as e:
        print(f"[!] Error Wayback: {e}")

    print(f"Historial web: https://web.archive.org/web/*/{dominio}")


# ---------------- GOOGLE DORKING ----------------

def google_dorking(dominio):
    print("\n[+] Generando búsquedas Google Dorking...")

    consultas = [
        ("Páginas indexadas", f"site:{dominio}"),
        ("Documentos PDF", f"site:{dominio} filetype:pdf"),
        ("Directorios indexados", f"site:{dominio} intitle:index.of"),
        ("Referencias en Google Drive",
         f'site:drive.google.com "{dominio}"'),
        ("Subdominios indexados", f"site:*.{dominio}")
    ]

    lineas = []

    for nombre, consulta in consultas:
        url = (
            "https://www.google.com/search?"
            + urllib.parse.urlencode({"q": consulta})
        )

        print(f"\n[{nombre}]")
        print(f"Consulta: {consulta}")
        print(f"URL: {url}")

        lineas.append(
            f"{nombre}\nConsulta: {consulta}\nURL: {url}\n"
        )

    guardar("google_dorks.txt", "\n".join(lineas))

    print("\n[+] Guardado en resultados/google_dorks.txt")
    print("[i] Las búsquedas se abren manualmente en el navegador.")


# ---------------- MENÚ PASIVO ----------------

def menu_pasiva(dominio):
    while True:
        print("""
╔══════════════════════════════════════╗
║       BUSC4 - ENUMERACIÓN PASIVA     ║
╠══════════════════════════════════════╣
║  [1] WHOIS                           ║
║  [2] Wayback Machine                 ║
║  [3] Google Dorking                  ║
║  [4] ENUMERACIÓN PASIVA COMPLETA     ║
║  [0] Volver al menú principal        ║
╚══════════════════════════════════════╝
""")

        opcion = input("BUSC4/PASIVA > ").strip()

        if opcion == "1":
            whois(dominio)

        elif opcion == "2":
            wayback(dominio)

        elif opcion == "3":
            google_dorking(dominio)

        elif opcion == "4":
            print("\n[+] Iniciando enumeración pasiva completa...")
            print("[1/3] WHOIS")
            whois(dominio)

            print("\n[2/3] Wayback Machine")
            wayback(dominio)

            print("\n[3/3] Google Dorking")
            google_dorking(dominio)

            print("\n[+] Enumeración pasiva completa finalizada.")

        elif opcion == "0":
            break

        else:
            print("[!] Opción no válida.")
            continue

        input("\nPulsa ENTER para volver al menú pasivo...")