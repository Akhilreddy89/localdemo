import numpy as np
import random

# Truck dimensions (53ft x 9ft x 8.5ft) converted to decimeters
TRUCK_LENGTH = int(53 * 3.048)
TRUCK_HEIGHT = int(9 * 3.048)
TRUCK_WIDTH = int(8.5 * 3.048)

truck_grid = np.zeros((TRUCK_LENGTH, TRUCK_HEIGHT, TRUCK_WIDTH), dtype=int)
placements = []
unplaced = []

# Package types
TEMPLATES = [
    {"type": "Fridge", "l": 6, "h": 15, "w": 6, "weight": 60, "fragile": 0},
    {"type": "Glass Door", "l": 8, "h": 15, "w": 2, "weight": 25, "fragile": 1},
    {"type": "LED TV", "l": 10, "h": 7, "w": 2, "weight": 20, "fragile": 1},
    {"type": "Bookshelf", "l": 12, "h": 10, "w": 4, "weight": 40, "fragile": 0},
    {"type": "Washing Mach", "l": 7, "h": 10, "w": 7, "weight": 55, "fragile": 0},
    {"type": "AC Unit", "l": 8, "h": 8, "w": 6, "weight": 45, "fragile": 0},
    {"type": "Desktop", "l": 6, "h": 6, "w": 4, "weight": 18, "fragile": 1},
    {"type": "Microwave", "l": 5, "h": 5, "w": 5, "weight": 22, "fragile": 1},
    {"type": "Drawer", "l": 8, "h": 9, "w": 4, "weight": 38, "fragile": 0},
    {"type": "Small Box", "l": 4, "h": 3, "w": 4, "weight": 5, "fragile": 0},
    {"type": "Oven", "l": 6, "h": 7, "w": 5, "weight": 30, "fragile": 1},
    {"type": "Filing Cab", "l": 8, "h": 10, "w": 5, "weight": 50, "fragile": 0}
]

# Generate 30 packages with LIFO delivery
packages = []
for i in range(30):
    template = random.choice(TEMPLATES)
    pkg = template.copy()
    pkg.update({
        "id": f"pkg{i+1:02d}",
        "delivery_order": 30 - i
    })
    packages.append(pkg)

packages.sort(key=lambda p: -p["delivery_order"])  # LIFO order

# Constraints
def is_supported(x, y, z, pkg):
    if y == 0: return True
    return all(truck_grid[x+dx, y-1, z+dz] != 0 for dx in range(pkg["l"]) for dz in range(pkg["w"]))

def not_on_fragile(x, y, z, pkg):
    if y == 0: return True
    for dx in range(pkg["l"]):
        for dz in range(pkg["w"]):
            below = truck_grid[x+dx, y-1, z+dz]
            if below:
                below_pkg = next((p for p in placements if p["index"] == below), None)
                if below_pkg and below_pkg["fragile"]:
                    return False
    return True

def not_heavier_than_below(x, y, z, pkg):
    if y == 0: return True
    for dx in range(pkg["l"]):
        for dz in range(pkg["w"]):
            below = truck_grid[x+dx, y-1, z+dz]
            if below:
                below_pkg = next((p for p in placements if p["index"] == below), None)
                if below_pkg and pkg["weight"] > below_pkg["weight"]:
                    return False
    return True

def is_within_bounds(x, y, z, pkg):
    return (x + pkg["l"] <= TRUCK_LENGTH and
            y + pkg["h"] <= TRUCK_HEIGHT and
            z + pkg["w"] <= TRUCK_WIDTH)

def is_free_space(x, y, z, pkg):
    return np.all(truck_grid[x:x+pkg["l"], y:y+pkg["h"], z:z+pkg["w"]] == 0)

# Attempt to place near center of truck for balance
def place_balanced(pkg, index):
    cx, cz = TRUCK_LENGTH // 2, TRUCK_WIDTH // 2
    x_range = sorted(range(0, TRUCK_LENGTH - pkg["l"] + 1), key=lambda x: abs(x - cx))
    z_range = sorted(range(0, TRUCK_WIDTH - pkg["w"] + 1), key=lambda z: abs(z - cz))
    y_range = range(0, TRUCK_HEIGHT - pkg["h"] + 1)

    for y in y_range:
        for x in x_range:
            for z in z_range:
                if (is_within_bounds(x, y, z, pkg) and is_free_space(x, y, z, pkg)
                    and is_supported(x, y, z, pkg)
                    and not_on_fragile(x, y, z, pkg)
                    and not_heavier_than_below(x, y, z, pkg)):
                    truck_grid[x:x+pkg["l"], y:y+pkg["h"], z:z+pkg["w"]] = index
                    placements.append({**pkg, "x": x, "y": y, "z": z, "index": index})
                    return True
    return False

# Place all packages
for i, pkg in enumerate(packages):
    if not place_balanced(pkg, i + 1):
        unplaced.append(pkg)

# Center of mass
def center_of_mass(placed):
    total_w = sum(p["weight"] for p in placed)
    cx = sum((p["x"] + p["l"]/2) * p["weight"] for p in placed) / total_w
    cy = sum((p["y"] + p["h"]/2) * p["weight"] for p in placed) / total_w
    cz = sum((p["z"] + p["w"]/2) * p["weight"] for p in placed) / total_w
    return round(cx, 2), round(cy, 2), round(cz, 2)

# Excel-style labeling
def excel_col(n):
    res = ""
    while n >= 0:
        res = chr(65 + (n % 26)) + res
        n = n // 26 - 1
    return res

def format_position(x, y, z):
    row = excel_col(x)
    col = str(z + 1)
    floor = str(y + 1)
    return f"{floor}{row}{col}"

# Output summary
print(f"🚛 Truck Size: {TRUCK_LENGTH} x {TRUCK_HEIGHT} x {TRUCK_WIDTH} dm")
print(f"📦 Total Packages: {len(packages)} | ✅ Placed: {len(placements)} | ❌ Unplaced: {len(unplaced)}")
print(f"🎯 Center of Mass: {center_of_mass(placements)}")
print(f"📌 Truck Center : {(TRUCK_LENGTH/2, TRUCK_HEIGHT/2, TRUCK_WIDTH/2)}")

print("\n📋 Placement Summary:")
sorted_placed = sorted(placements, key=lambda p: p["delivery_order"])
for i, pkg in enumerate(sorted_placed, 1):
    print(f"{i:02d}. {pkg['type']:12s} → {format_position(pkg['x'], pkg['y'], pkg['z'])}")
