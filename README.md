# SuggestionJS
This is free to use code made by @TrixNEW and @EncoyIX
This is one of the most advanced suggestion code out to free use right now.
Please report any bugs or errors into the repo.

# config.js
{
  "suggestionChannelId": "ENTER_CHANNEL_ID",
  "approverRoleId": "ENTER_APPROVED_ROLE_ID"
}

# Index.js
const {
  Client,
  GatewayIntentBits,
  EmbedBuilder,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  ModalBuilder,
  TextInputBuilder,
  TextInputStyle,
} = require("discord.js");
const config = require("./config.json");
const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
    GatewayIntentBits.GuildMessageReactions,
  ],
});

const suggestionChannelId = config.suggestionChannelId;
const approverRoleId = config.approverRoleId;
const userVotes = {};

client.once("ready", () => {
  console.log("Bot is online!");
  console.log("https://discord.gg/nVYk2S39b4");
});

client.on("messageCreate", (message) => {
  if (message.content.startsWith('!suggest')) {
    const suggestion = message.content.slice('!suggest '.length);
    const suggestionChannel = client.channels.cache.get(suggestionChannelId);

    if (!suggestionChannel) {
      console.error(`Suggestion channel not found with ID: ${suggestionChannelId}`);
      return;
    }

    const suggestionEmbed = new EmbedBuilder()
      .setColor(0x00b2ff)
      .setTitle("📝 New proposal")
      .setDescription(`**Suggestion :**\n\`\`\`${suggestion}\`\`\``)
      .setTimestamp()
      .setFooter({ text: `Sent by : ${message.author.tag}` })
      .setThumbnail(message.author.displayAvatarURL())
      .addFields(
        { name: "the condition", value: "⏳ pending", inline: true },
        { name: "the support", value: "👍 0 | 👎 0", inline: true },
      );
    const row = new ActionRowBuilder().addComponents(
      new ButtonBuilder()
        .setCustomId(`accept_${message.author.id}`)
        .setLabel("Accept")
        .setStyle(ButtonStyle.Success),
      new ButtonBuilder()
        .setCustomId(`reject_${message.author.id}`)
        .setLabel("Refuse")
        .setStyle(ButtonStyle.Danger),
      new ButtonBuilder()
        .setCustomId("upvote")
        .setLabel("👍")
        .setStyle(ButtonStyle.Primary),
      new ButtonBuilder()
        .setCustomId("downvote")
        .setLabel("👎")
        .setStyle(ButtonStyle.Primary),
    );

    suggestionChannel
      .send({ embeds: [suggestionEmbed], components: [row] })
      .then(() => message.delete())
      .catch(console.error);
  } 
});

client.on("interactionCreate", async (interaction) => {
  if (!interaction.isButton()) return;

  // Handle expired interaction
  if (Date.now() - interaction.createdTimestamp > 60000) {
    await interaction.reply({ content: 'This interaction has expired.', ephemeral: true }); 
    return; 
  }

  const messageId = interaction.message.id;
  const userId = interaction.user.id;

  if (
    interaction.customId.startsWith("accept") ||
    interaction.customId.startsWith("reject")
  ) {
    const roleId = approverRoleId;
    if (!interaction.member.roles.cache.has(roleId)) {
      return interaction.reply({
        content: "You do not have permission to use this button.",
        ephemeral: true,
      });
    }

    const modal = new ModalBuilder()
      .setCustomId(`response-modal-${interaction.customId}`)
      .setTitle("Response");

    const reasonInput = new TextInputBuilder()
      .setCustomId("reason")
      .setLabel("Reason")
      .setStyle(TextInputStyle.Paragraph);

    const actionRow = new ActionRowBuilder().addComponents(reasonInput);

    modal.addComponents(actionRow);

    await interaction.showModal(modal);
  } else if (
    interaction.customId === "upvote" ||
    interaction.customId === "downvote"
  ) {
    if (!userVotes[messageId]) userVotes[messageId] = new Set();
    if (userVotes[messageId].has(userId)) {
      return interaction.reply({
        content: "You have already voted on this proposal .",
        ephemeral: true,
      });
    }
    userVotes[messageId].add(userId);

    const originalEmbed = interaction.message.embeds[0];
    const fields = originalEmbed.fields;
    let upvotes = parseInt(fields[1].value.split("|")[0].trim().split(" ")[1]);
    let downvotes = parseInt(
      fields[1].value.split("|")[1].trim().split(" ")[1],
    );

    if (interaction.customId === "upvote") upvotes++;
    if (interaction.customId === "downvote") downvotes++;

    const updatedEmbed = new EmbedBuilder(originalEmbed).spliceFields(1, 1, {
      name: "the support",
      value: `👍 ${upvotes} | 👎 ${downvotes}`,
      inline: true,
    });

    await interaction.update({
      embeds: [updatedEmbed],
      components: interaction.message.components,
    });
  }
});

client.on("interactionCreate", async (interaction) => {
  if (interaction.isModalSubmit()) {
    const reason = interaction.fields.getTextInputValue("reason");
    const originalEmbed = interaction.message.embeds[0];
    const decision = interaction.customId.includes("accept")
      ? "It has been accepted"
      : "access denied";

    const acceptedColor = 0x28a745;
    const rejectedColor = 0xdc3545;

    const color = decision === "It has been accepted" ? acceptedColor : rejectedColor;

    const updatedButtons = new ActionRowBuilder().addComponents(
      new ButtonBuilder()
        .setCustomId("upvote")
        .setLabel("👍")
        .setStyle(ButtonStyle.Primary),
      new ButtonBuilder()
        .setCustomId("downvote")
        .setLabel("👎")
        .setStyle(ButtonStyle.Primary),
    );

    const updatedEmbed = new EmbedBuilder(originalEmbed)
      .spliceFields(0, 1, { name: decision, value: reason, inline: true })
      .setColor(color);

    await interaction.message.edit({
      embeds: [updatedEmbed],
      components: [updatedButtons],
    });

    await interaction.reply({
      content: `The suggestion has been ${decision.toLowerCase()}.`,
      ephemeral: true,
    });

    const user = await interaction.guild.members.fetch(
      interaction.customId.split("_")[1],
    );

    if (user) {
      user.send({ content: `Your suggestion was answered with ${decision}` });
    }
  }
});



client.login(process.env.TOKEN);

# Token
Enter your bots token anywhere where it says 'TOKEN'

# package.json
{
    "name": "suggestions-bot",
    "version": "1.0.0",
    "description": "A Discord bot for handling suggestions",
    "main": "index.js",
    "scripts": {
      "start": "node index.js"
    },
    "author": "Trix",
    "license": "MIT",
    "dependencies": {
      "discord.js": "^14.0.0"
    }
  }
  
