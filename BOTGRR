# -*- coding: utf-8 -*-
import json
import os
from datetime import datetime

import discord
from discord.ext import commands
from discord import app_commands
from discord.ui import View, Modal, TextInput, Button

import os
TOKEN = os.getenv("TOKEN")

PREFIXO_NICK = "ESTG.GRR"
COR_AZUL_ESCURO = 0x0A2A66
CONFIG_FILE = "config.json"

intents = discord.Intents.default()
intents.message_content = True
intents.members = True
intents.guilds = True

bot = commands.Bot(command_prefix="!", intents=intents)


# ===========================
# JSON
# ===========================
def load_json(file_path: str) -> dict:
    if not os.path.exists(file_path):
        return {}
    try:
        with open(file_path, "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception:
        return {}


def save_json(file_path: str, data: dict) -> None:
    with open(file_path, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=4, ensure_ascii=False)


def get_config() -> dict:
    return load_json(CONFIG_FILE)


def save_config(data: dict) -> None:
    save_json(CONFIG_FILE, data)


# ===========================
# FUNÇÕES AUXILIARES
# ===========================
def has_approval_permission(member: discord.Member, role_ids: list[int]) -> bool:
    return any(role.id in role_ids for role in member.roles)


def format_role_mentions(role_ids: list[int]) -> str:
    return " ".join(f"<@&{role_id}>" for role_id in role_ids if role_id)


def get_guild_config(guild_id: int) -> dict | None:
    return get_config().get(str(guild_id))


def ensure_guild_config(guild_id: int) -> dict:
    data = get_config()
    key = str(guild_id)
    if key not in data:
        data[key] = {
            "canal_solicitacoes": None,
            "canal_logs": None,
            "cargo_set": None,
            "aprovadores": [],
            "panel_channel_id": None,
            "panel_message_id": None
        }
    return data


# ===========================
# VIEW DE APROVAÇÃO
# ===========================
class AprovarView(View):
    def __init__(self, guild_id: int, user_id: int, nome: str, passaporte: str):
        super().__init__(timeout=None)
        self.guild_id = guild_id
        self.user_id = user_id
        self.nome = nome
        self.passaporte = passaporte

    @discord.ui.button(
        label="Aprovar",
        style=discord.ButtonStyle.success,
        custom_id="aprovar_btn"
    )
    async def aprovar(self, interaction: discord.Interaction, button: Button):
        guild = interaction.guild
        if guild is None:
            await interaction.response.send_message("❌ Servidor não encontrado.", ephemeral=True)
            return

        config = get_guild_config(guild.id)
        if not config:
            await interaction.response.send_message("❌ Este servidor não está configurado.", ephemeral=True)
            return

        if not has_approval_permission(interaction.user, config.get("aprovadores", [])):
            await interaction.response.send_message("❌ Você não tem permissão para aprovar.", ephemeral=True)
            return

        member = guild.get_member(self.user_id)
        if member is None:
            await interaction.response.send_message("❌ Usuário não encontrado no servidor.", ephemeral=True)
            return

        cargo_set_id = config.get("cargo_set")
        role = guild.get_role(cargo_set_id) if cargo_set_id else None
        if role is None:
            await interaction.response.send_message("❌ O cargo configurado não foi encontrado.", ephemeral=True)
            return

        try:
            await member.add_roles(role, reason="Solicitação aprovada")
        except discord.Forbidden:
            await interaction.response.send_message(
                "❌ Não consegui adicionar o cargo. Verifique a hierarquia e permissões do bot.",
                ephemeral=True
            )
            return

        nick_final = f"[{PREFIXO_NICK}] {self.nome.upper()} | {self.passaporte}"
        status_nick = "✅ Nick alterado com sucesso."

        try:
            await member.edit(nick=nick_final, reason="Solicitação aprovada")
        except discord.Forbidden:
            status_nick = "⚠️ Cargo entregue, mas não consegui alterar o nick."

        logs_channel_id = config.get("canal_logs")
        logs_channel = guild.get_channel(logs_channel_id) if logs_channel_id else None

        if isinstance(logs_channel, discord.TextChannel):
            embed_logs = discord.Embed(
                title="✅ Solicitação Aprovada",
                color=COR_AZUL_ESCURO,
                timestamp=datetime.now()
            )
            embed_logs.add_field(name="Usuário", value=member.mention, inline=False)
            embed_logs.add_field(name="Nome", value=self.nome.upper(), inline=False)
            embed_logs.add_field(name="Passaporte", value=self.passaporte, inline=False)
            embed_logs.add_field(name="Cargo", value=role.mention, inline=False)
            embed_logs.add_field(name="Nick final", value=nick_final, inline=False)
            embed_logs.add_field(name="Status do nick", value=status_nick, inline=False)
            embed_logs.add_field(name="Aprovado por", value=interaction.user.mention, inline=False)
            await logs_channel.send(embed=embed_logs)

        for child in self.children:
            child.disabled = True

        embed_final = discord.Embed(
            title="✅ Solicitação Finalizada",
            description=(
                f"**Usuário:** {member.mention}\n"
                f"**Nome:** {self.nome.upper()}\n"
                f"**Passaporte:** {self.passaporte}\n"
                f"**Resultado:** Aprovado\n"
                f"**Aprovado por:** {interaction.user.mention}\n\n"
                f"{status_nick}"
            ),
            color=COR_AZUL_ESCURO,
            timestamp=datetime.now()
        )

        await interaction.response.edit_message(embed=embed_final, view=self)

    @discord.ui.button(
        label="Negar",
        style=discord.ButtonStyle.danger,
        custom_id="negar_btn"
    )
    async def negar(self, interaction: discord.Interaction, button: Button):
        guild = interaction.guild
        if guild is None:
            await interaction.response.send_message("❌ Servidor não encontrado.", ephemeral=True)
            return

        config = get_guild_config(guild.id)
        if not config:
            await interaction.response.send_message("❌ Este servidor não está configurado.", ephemeral=True)
            return

        if not has_approval_permission(interaction.user, config.get("aprovadores", [])):
            await interaction.response.send_message("❌ Você não tem permissão para negar.", ephemeral=True)
            return

        member = guild.get_member(self.user_id)
        logs_channel_id = config.get("canal_logs")
        logs_channel = guild.get_channel(logs_channel_id) if logs_channel_id else None

        if isinstance(logs_channel, discord.TextChannel):
            embed_logs = discord.Embed(
                title="❌ Solicitação Negada",
                color=COR_AZUL_ESCURO,
                timestamp=datetime.now()
            )
            embed_logs.add_field(
                name="Usuário",
                value=member.mention if member else f"ID: {self.user_id}",
                inline=False
            )
            embed_logs.add_field(name="Nome", value=self.nome.upper(), inline=False)
            embed_logs.add_field(name="Passaporte", value=self.passaporte, inline=False)
            embed_logs.add_field(name="Negado por", value=interaction.user.mention, inline=False)
            await logs_channel.send(embed=embed_logs)

        for child in self.children:
            child.disabled = True

        embed_final = discord.Embed(
            title="❌ Solicitação Finalizada",
            description=(
                f"**Usuário:** {member.mention if member else f'ID: {self.user_id}'}\n"
                f"**Nome:** {self.nome.upper()}\n"
                f"**Passaporte:** {self.passaporte}\n"
                f"**Resultado:** Negado\n"
                f"**Negado por:** {interaction.user.mention}"
            ),
            color=COR_AZUL_ESCURO,
            timestamp=datetime.now()
        )

        await interaction.response.edit_message(embed=embed_final, view=self)


# ===========================
# MODAL
# ===========================
class SetModal(Modal, title="Solicitação"):
    nome = TextInput(
        label="Nome",
        placeholder="Digite o nome que vai ficar no nick",
        required=True,
        max_length=20
    )

    passaporte = TextInput(
        label="Passaporte",
        placeholder="Somente números",
        required=True,
        max_length=10
    )

    def __init__(self, user_id: int):
        super().__init__()
        self.user_id = user_id

    async def on_submit(self, interaction: discord.Interaction):
        if not self.passaporte.value.isdigit():
            await interaction.response.send_message("❌ O passaporte deve conter apenas números.", ephemeral=True)
            return

        guild = interaction.guild
        if guild is None:
            await interaction.response.send_message("❌ Servidor não encontrado.", ephemeral=True)
            return

        config = get_guild_config(guild.id)
        if not config:
            await interaction.response.send_message("❌ Este servidor não está configurado. Use /configurar.", ephemeral=True)
            return

        solicitacoes_channel_id = config.get("canal_solicitacoes")
        solicitacoes_channel = guild.get_channel(solicitacoes_channel_id) if solicitacoes_channel_id else None

        if not isinstance(solicitacoes_channel, discord.TextChannel):
            await interaction.response.send_message("❌ O canal de solicitações configurado não foi encontrado.", ephemeral=True)
            return

        mencoes = format_role_mentions(config.get("aprovadores", []))

        embed = discord.Embed(
            title="📋 Nova Solicitação",
            color=COR_AZUL_ESCURO,
            timestamp=datetime.now()
        )
        embed.add_field(name="Usuário", value=interaction.user.mention, inline=False)
        embed.add_field(name="Nome", value=self.nome.value.upper(), inline=False)
        embed.add_field(name="Passaporte", value=self.passaporte.value, inline=False)
        embed.set_footer(text="Aguardando aprovação")

        await solicitacoes_channel.send(
            content=f"{mencoes}\nNova solicitação aguardando aprovação." if mencoes else "Nova solicitação aguardando aprovação.",
            embed=embed,
            view=AprovarView(
                guild_id=guild.id,
                user_id=self.user_id,
                nome=self.nome.value,
                passaporte=self.passaporte.value
            )
        )

        await interaction.response.send_message("✅ Solicitação enviada com sucesso.", ephemeral=True)


# ===========================
# VIEW DO PAINEL
# ===========================
class PainelView(View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(
        label="Pedir Set",
        style=discord.ButtonStyle.primary,
        custom_id="pedir_set_btn"
    )
    async def abrir(self, interaction: discord.Interaction, button: Button):
        await interaction.response.send_modal(SetModal(interaction.user.id))


# ===========================
# SLASH COMMANDS
# ===========================
@bot.tree.command(name="configurar", description="Configura os canais e o cargo do set.")
@app_commands.describe(
    canal_solicitacoes="Canal onde as solicitações serão enviadas",
    canal_logs="Canal onde os logs serão enviados",
    cargo_set="Cargo que será entregue ao aprovar"
)
@app_commands.checks.has_permissions(manage_guild=True)
async def configurar(
    interaction: discord.Interaction,
    canal_solicitacoes: discord.TextChannel,
    canal_logs: discord.TextChannel,
    cargo_set: discord.Role
):
    guild = interaction.guild
    if guild is None:
        await interaction.response.send_message("❌ Este comando só pode ser usado em servidor.", ephemeral=True)
        return

    data = ensure_guild_config(guild.id)
    key = str(guild.id)

    data[key]["canal_solicitacoes"] = canal_solicitacoes.id
    data[key]["canal_logs"] = canal_logs.id
    data[key]["cargo_set"] = cargo_set.id

    save_config(data)

    embed = discord.Embed(
        title="✅ Servidor Configurado",
        color=COR_AZUL_ESCURO,
        timestamp=datetime.now()
    )
    embed.add_field(name="Canal de Solicitações", value=canal_solicitacoes.mention, inline=False)
    embed.add_field(name="Canal de Logs", value=canal_logs.mention, inline=False)
    embed.add_field(name="Cargo do Set", value=cargo_set.mention, inline=False)
    embed.add_field(
        name="Cargos Aprovadores",
        value=format_role_mentions(data[key].get("aprovadores", [])) or "Nenhum configurado ainda",
        inline=False
    )

    await interaction.response.send_message(embed=embed, ephemeral=True)


@bot.tree.command(name="addaprovador", description="Adiciona um cargo aprovador.")
@app_commands.describe(cargo="Cargo que poderá aprovar e negar solicitações")
@app_commands.checks.has_permissions(manage_guild=True)
async def addaprovador(interaction: discord.Interaction, cargo: discord.Role):
    guild = interaction.guild
    if guild is None:
        await interaction.response.send_message("❌ Este comando só pode ser usado em servidor.", ephemeral=True)
        return

    data = ensure_guild_config(guild.id)
    key = str(guild.id)

    aprovadores = data[key].get("aprovadores", [])
    if cargo.id in aprovadores:
        await interaction.response.send_message("⚠️ Esse cargo já está na lista de aprovadores.", ephemeral=True)
        return

    aprovadores.append(cargo.id)
    data[key]["aprovadores"] = aprovadores
    save_config(data)

    await interaction.response.send_message(
        f"✅ Cargo {cargo.mention} adicionado como aprovador.",
        ephemeral=True
    )


@bot.tree.command(name="removeraprovador", description="Remove um cargo aprovador.")
@app_commands.describe(cargo="Cargo que será removido da lista de aprovadores")
@app_commands.checks.has_permissions(manage_guild=True)
async def removeraprovador(interaction: discord.Interaction, cargo: discord.Role):
    guild = interaction.guild
    if guild is None:
        await interaction.response.send_message("❌ Este comando só pode ser usado em servidor.", ephemeral=True)
        return

    config = get_guild_config(guild.id)
    if not config:
        await interaction.response.send_message("❌ Este servidor ainda não foi configurado.", ephemeral=True)
        return

    aprovadores = config.get("aprovadores", [])
    if cargo.id not in aprovadores:
        await interaction.response.send_message("⚠️ Esse cargo não está na lista de aprovadores.", ephemeral=True)
        return

    aprovadores.remove(cargo.id)
    data = get_config()
    data[str(guild.id)]["aprovadores"] = aprovadores
    save_config(data)

    await interaction.response.send_message(
        f"✅ Cargo {cargo.mention} removido da lista de aprovadores.",
        ephemeral=True
    )


@bot.tree.command(name="painel", description="Envia o painel de solicitação de set neste canal.")
@app_commands.checks.has_permissions(manage_guild=True)
async def painel(interaction: discord.Interaction):
    guild = interaction.guild
    channel = interaction.channel

    if guild is None or not isinstance(channel, discord.TextChannel):
        await interaction.response.send_message("❌ Este comando só pode ser usado em canal de texto.", ephemeral=True)
        return

    config = get_guild_config(guild.id)
    if not config:
        await interaction.response.send_message("❌ Este servidor não está configurado. Use /configurar primeiro.", ephemeral=True)
        return

    panel_channel_id = config.get("panel_channel_id")
    panel_message_id = config.get("panel_message_id")

    if panel_channel_id and panel_message_id:
        old_channel = guild.get_channel(panel_channel_id)
        if isinstance(old_channel, discord.TextChannel):
            try:
                old_message = await old_channel.fetch_message(panel_message_id)
                await old_message.delete()
            except (discord.NotFound, discord.Forbidden, discord.HTTPException):
                pass

    embed = discord.Embed(
        title="👮 Painel de Solicitação de Set - GRR",
        description=(
            "🔹 **Grupo De Respostas Rapidas 👮**\n\n"
            "🔹 Preencha as informações com atenção:\n"
            "• Nome\n"
            "• Passaporte"
        ),
        color=COR_AZUL_ESCURO
    )
    embed.set_footer(text="Raul System")

    await interaction.response.send_message("✅ Painel enviado com sucesso.", ephemeral=True)

    msg = await channel.send(embed=embed, view=PainelView())

    data = get_config()
    data[str(guild.id)]["panel_channel_id"] = channel.id
    data[str(guild.id)]["panel_message_id"] = msg.id
    save_config(data)


@bot.tree.command(name="verconfig", description="Mostra a configuração atual do servidor.")
async def verconfig(interaction: discord.Interaction):
    guild = interaction.guild
    if guild is None:
        await interaction.response.send_message("❌ Este comando só pode ser usado em servidor.", ephemeral=True)
        return

    config = get_guild_config(guild.id)
    if not config:
        await interaction.response.send_message("❌ Este servidor ainda não foi configurado.", ephemeral=True)
        return

    canal_solicitacoes = guild.get_channel(config["canal_solicitacoes"]) if config.get("canal_solicitacoes") else None
    canal_logs = guild.get_channel(config["canal_logs"]) if config.get("canal_logs") else None
    cargo_set = guild.get_role(config["cargo_set"]) if config.get("cargo_set") else None

    embed = discord.Embed(
        title="⚙️ Configuração do Servidor",
        color=COR_AZUL_ESCURO,
        timestamp=datetime.now()
    )
    embed.add_field(
        name="Canal de Solicitações",
        value=canal_solicitacoes.mention if canal_solicitacoes else "Não encontrado",
        inline=False
    )
    embed.add_field(
        name="Canal de Logs",
        value=canal_logs.mention if canal_logs else "Não encontrado",
        inline=False
    )
    embed.add_field(
        name="Cargo do Set",
        value=cargo_set.mention if cargo_set else "Não encontrado",
        inline=False
    )
    embed.add_field(
        name="Cargos Aprovadores",
        value=format_role_mentions(config.get("aprovadores", [])) or "Nenhum",
        inline=False
    )

    await interaction.response.send_message(embed=embed, ephemeral=True)


@bot.tree.command(name="resetconfig", description="Reseta a configuração deste servidor.")
@app_commands.checks.has_permissions(manage_guild=True)
async def resetconfig(interaction: discord.Interaction):
    guild = interaction.guild
    if guild is None:
        await interaction.response.send_message("❌ Este comando só pode ser usado em servidor.", ephemeral=True)
        return

    data = get_config()
    key = str(guild.id)

    if key not in data:
        await interaction.response.send_message("⚠️ Este servidor não possui configuração salva.", ephemeral=True)
        return

    panel_channel_id = data[key].get("panel_channel_id")
    panel_message_id = data[key].get("panel_message_id")

    if panel_channel_id and panel_message_id:
        channel = guild.get_channel(panel_channel_id)
        if isinstance(channel, discord.TextChannel):
            try:
                msg = await channel.fetch_message(panel_message_id)
                await msg.delete()
            except (discord.NotFound, discord.Forbidden, discord.HTTPException):
                pass

    del data[key]
    save_config(data)

    await interaction.response.send_message("🧹 Configuração do servidor resetada com sucesso.", ephemeral=True)


# ===========================
# ERROS DOS SLASH COMMANDS
# ===========================
@configurar.error
@addaprovador.error
@removeraprovador.error
@painel.error
@resetconfig.error
async def admin_command_error(interaction: discord.Interaction, error: app_commands.AppCommandError):
    if isinstance(error, app_commands.errors.MissingPermissions):
        if interaction.response.is_done():
            await interaction.followup.send(
                "❌ Você precisa da permissão **Gerenciar Servidor** para usar esse comando.",
                ephemeral=True
            )
        else:
            await interaction.response.send_message(
                "❌ Você precisa da permissão **Gerenciar Servidor** para usar esse comando.",
                ephemeral=True
            )
        return

    if interaction.response.is_done():
        await interaction.followup.send("❌ Ocorreu um erro ao executar o comando.", ephemeral=True)
    else:
        await interaction.response.send_message("❌ Ocorreu um erro ao executar o comando.", ephemeral=True)


# ===========================
# INICIALIZAÇÃO
# ===========================
@bot.event
async def setup_hook():
    bot.add_view(PainelView())
    bot.add_view(AprovarView(guild_id=0, user_id=0, nome="NOME", passaporte="000"))
    await bot.tree.sync()


@bot.event
async def on_ready():
    print(f"✅ Bot conectado como {bot.user}")


# ===========================
# INICIAR BOT
# ===========================
if not TOKEN or TOKEN == "COLE_SEU_TOKEN_AQUI":
    raise ValueError("Defina um token válido na variável TOKEN.")

bot.run(TOKEN)
