# miami-roleplay-BOT
# Al inicio, agrega estos imports si no están
import json
import os
from datetime import datetime, timedelta
import random

# Archivos
DATA_FILE = 'economy.json'
ROLE_INCOME_FILE = 'role_income.json'
SHOP_FILE = 'shop_items.json'  # Nuevo para la tienda

# Cargar / guardar shop (por guild)
def load_shop():
    if os.path.exists(SHOP_FILE):
        with open(SHOP_FILE, 'r') as f:
            return json.load(f)
    return {}

def save_shop(data):
    with open(SHOP_FILE, 'w') as f:
        json.dump(data, f, indent=4)

shop_data = load_shop()

# Obtener shop de un guild
def get_shop(guild_id):
    guild_str = str(guild_id)
    if guild_str not in shop_data:
        shop_data[guild_str] = {}  # Ej: {"Coffee": {"price": 50, "desc": "Una taza caliente"}, ...}
    return shop_data[guild_str]

# En get_user (agrega 'inventory' si no existe)
def get_user(guild_id, user_id):
    guild_str = str(guild_id)
    user_str = str(user_id)
    if guild_str not in data:
        data[guild_str] = {}
    if user_str not in data[guild_str]:
        data[guild_str][user_str] = {
            'cash': 1000,
            'bank': 0,
            'last_work': None,
            'last_crime': None,
            'last_rob': None,
            'last_daily': None,
            'last_collect': None,
            'inventory': {}  # Nuevo: {"Coffee": 3, "Role Police": 1, ...}
        }
    return data[guild_str][user_str]

# Comando /shop - Lista simple
@bot.tree.command(name="shop", description="Mira la tienda de Liberty County")
async def shop(interaction: discord.Interaction):
    guild_shop = get_shop(interaction.guild_id)
    if not guild_shop:
        await interaction.response.send_message("La tienda está vacía por ahora. ¡Vuelve pronto! 🏪", ephemeral=True)
        return
    
    embed = discord.Embed(title="🏪 Tienda de Liberty County - Miami Roleplay", color=0xffa500)
    embed.description = "Compra items con /buy <nombre del item>"
    
    for item_name, info in guild_shop.items():
        price = info.get('price', 0)
        desc = info.get('desc', 'Sin descripción')
        embed.add_field(
            name=f"{item_name} - ${price:,}",
            value=desc,
            inline=False
        )
    
    embed.set_footer(text="Items disponibles para compra inmediata 🌴")
    await interaction.response.send_message(embed=embed)

# Comando /buy <item>
@bot.tree.command(name="buy", description="Compra un item de la tienda")
@app_commands.describe(item="Nombre exacto del item a comprar")
async def buy(interaction: discord.Interaction, item: str):
    guild_shop = get_shop(interaction.guild_id)
    item = item.strip()  # Limpia espacios
    
    if item not in guild_shop:
        await interaction.response.send_message(f"El item **{item}** no está en la tienda. Usa /shop para ver disponibles.", ephemeral=True)
        return
    
    info = guild_shop[item]
    price = info['price']
    
    user = get_user(interaction.guild_id, interaction.user.id)
    if user['cash'] < price:
        await interaction.response.send_message(f"No tienes suficiente cash. Necesitas ${price:,}, tienes ${user['cash']:,}.", ephemeral=True)
        return
    
    user['cash'] -= price
    
    # Agregar al inventory
    inv = user['inventory']
    if item in inv:
        inv[item] += 1
    else:
        inv[item] = 1
    
    save_data(data)
    await interaction.response.send_message(f"¡Compraste **{item}** por **${price:,}**! Disfrútalo en las calles de Miami. 💸\nNuevo cash: **${user['cash']:,}**")

# Comando /inventory
@bot.tree.command(name="inventory", description="Mira tu inventario de items")
@app_commands.describe(member="Usuario (opcional, deja vacío para ti)")
async def inventory(interaction: discord.Interaction, member: discord.Member = None):
    target = member if member else interaction.user
    user = get_user(interaction.guild_id, target.id)
    
    inv = user.get('inventory', {})
    if not inv:
        await interaction.response.send_message(f"El inventario de {target.name} está vacío. ¡Ve de compras con /shop! 🛒", ephemeral=(target != interaction.user))
        return
    
    embed = discord.Embed(title=f"🎒 Inventario de {target.name} - Liberty County", color=0x00bfff)
    for item, qty in inv.items():
        embed.add_field(name=item, value=f"Cantidad: **{qty}**", inline=True)
    
    embed.set_footer(text=f"Cash: ${user['cash']:,} | Bank: ${user['bank']:,}")
    await interaction.response.send_message(embed=embed)

# Al final, después de on_ready, sincroniza comandos
@bot.event
async def on_ready():
    print(f'Bot conectado como {bot.user}')
    try:
        synced = await bot.tree.sync()
        print(f'Sincronizados {len(synced)} comandos.')
    except Exception as e:
        print(e)

# Para probar la tienda: Edita manualmente shop_items.json (crea el archivo si no existe)
# Ejemplo de contenido inicial (pégalo en shop_items.json):
# {
#   "Coffee": {"price": 50, "desc": "Una taza de café para mantenerte despierto en patrulla"},
#   "Police Badge": {"price": 5000, "desc": "Insignia oficial - rol estético"},
#   "Pizza": {"price": 150, "desc": "Pizza de Miami para recargar energía"}
# }

bot.run(os.getenv('BOT_TOKEN'))
