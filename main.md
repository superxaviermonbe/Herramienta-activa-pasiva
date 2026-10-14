import re
from pathlib import Path

import pasiva
import activa

CYAN = "\033[96m"
GREEN = "\033[92m"
RED = "\033[91m"
RESET = "\033[0m"
BOLD = "\033[1m"


def banner():
    print(CYAN + BOLD + r"""
 ____  _   _ ____   ____ _  _
| __ )| | | / ___| / ___| || |
|  _ \| | | \___ \| |   | || |_
| |_) | |_| |___) | |___|__   _|
|____/ \___/|____/ \____|  |_|
""" + RESET)

    print(GREEN + "       BUSC4 // RECON FRAMEWORK v2.0" + RESET)
    print("       Passive & Active Enumeration")
    print("-" * 45)


def validar_dominio(dominio):
    dominio = dominio.strip().lower().rstrip(".")

    if not dominio or len(dominio) > 253 or "." not in dominio:
        return False

    etiquetas = dominio.split(".")

    return all(
        len(e) <= 63
        and re.fullmatch(
            r"[a-z0-9](?:[a-z0-9-]*[a-z0-9])?",
            e
        )
        for e in etiquetas
    )


def ver_resultados():
    carpeta = Path("resultados")
    carpeta.mkdir(exist_ok=True)

    archivos = sorted(carpeta.iterdir())

    print("\n========== RESULTADOS ==========")

    if not archivos:
        print("Todavía no hay resultados guardados.")
    else:
        for archivo in archivos:
            if archivo.is_file():
                print(f"- {archivo.name} ({archivo.stat().st_size} bytes)")

    print(f"\nRuta: {carpeta.resolve()}")


def main():
    Path("resultados").mkdir(exist_ok=True)

    banner()

    dominio = input("Introduce el dominio objetivo: ").strip()

    if not validar_dominio(dominio):
        print(RED + "[!] Dominio no válido." + RESET)
        return

    dominio = dominio.lower().rstrip(".")

    while True:
        print(f"""
{CYAN}{BOLD}========== MENÚ PRINCIPAL =========={RESET}
Objetivo: {dominio}

[1] Enumeración pasiva
[2] Enumeración activa
[3] Ver resultados guardados
[4] Cambiar dominio
[0] Salir
""")

        opcion = input("BUSC4 > ").strip()

        if opcion == "1":
            pasiva.menu_pasiva(dominio)

        elif opcion == "2":
            activa.menu_activa(dominio)

        elif opcion == "3":
            ver_resultados()

        elif opcion == "4":
            nuevo = input("Nuevo dominio: ").strip()

            if validar_dominio(nuevo):
                dominio = nuevo.lower().rstrip(".")
                print("[+] Objetivo actualizado.")
            else:
                print(RED + "[!] Dominio no válido." + RESET)

        elif opcion == "0":
            print(GREEN + "[+] Cerrando BUSC4." + RESET)
            break

        else:
            print(RED + "[!] Opción no válida." + RESET)


if __name__ == "__main__":
    main()