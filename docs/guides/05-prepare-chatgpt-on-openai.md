# Prepare ChatGPT on OpenAI

Verified with TaskWeave 0.2.0rc1, tunnel-client 0.0.19, on 2026-11-03

## Goal and time

Do the OpenAI-side work that TaskWeave cannot do for you: create a tunnel, get its ID and a runtime key, get the permissions, and make sure your ChatGPT plan and workspace allow a custom connector. About 20 minutes, plus a possible wait of up to 30 minutes for a role change.

## You need

- Access to the OpenAI Platform organization that will own the tunnel.
- Permission to create a tunnel, or an owner or RBAC administrator who can do it and grant **Tunnels Read + Use** (and **Tunnels Read + Manage** if you will edit the tunnel).
- A ChatGPT workspace and plan that allow custom MCP connectors. OpenAI and your workspace administrator decide this; check the official documentation.
- A password manager or another safe place for the runtime key while you work. Do not put it in a file or a chat.

## Steps

1. Open the OpenAI Platform's tunnel management page and create a tunnel, or open the tunnel your organization already made for you. The **tunnel ID** is shown there. It is private configuration, but it is not the secret that authenticates the connection.

2. Create a **runtime key** for the tunnel client. Use a runtime key, not an admin key. The runtime key is the secret; TaskWeave never stores it, and the start script asks for it in its own window.

3. Ask an owner or RBAC administrator for **Tunnels Read + Use**. If you will create or edit tunnels, you also need **Tunnels Read + Manage**. A new role can take up to 30 minutes to take effect.

4. While you wait, confirm that your ChatGPT workspace allows custom MCP connectors. The option may be called **Connectors**, **Apps** or **Plugins**, and the workspace administrator may need to enable it.

5. Confirm that the tunnel is associated with that ChatGPT workspace, not only with a personal Platform organization. A tunnel tied only to a personal organization does not appear in a business workspace.

6. Check the tunnel client version you will install. Get the latest release from the official OpenAI tunnel-client repository; note the version and keep the file somewhere safe. The Connect page and [Connect ChatGPT](06-connect-chatgpt.md) show the checks it will run.

7. Write a short tick list for yourself: tunnel ID, runtime key in the password manager, **Tunnels Read + Use** granted, **Tunnels Read + Manage** if needed, plan allows custom connectors, tunnel associated with the workspace. Do not put the key in the list.

## You should see

- A tunnel with an ID in the OpenAI Platform.
- A runtime key you can retrieve from the password manager.
- The role shown as granted, or a clear note about the 30-minute wait.
- The custom-connector option in the ChatGPT workspace, or a clear statement that an administrator must enable it.
- The tunnel associated with the right workspace.

## If it does not match

| What you see | What to do |
|---|---|
| You cannot create a tunnel | Ask an organization owner or RBAC administrator. You need the *Tunnels* permissions. |
| `Tunnels access required` in ChatGPT | Confirm the Platform organization and ask for **Tunnels Read + Use**. Role changes can take up to 30 minutes. |
| Your tunnel is not in ChatGPT's list | The tunnel is tied to another workspace or organization. Add the workspace in the Platform tunnel settings, or ask an administrator. |
| No option to add a custom MCP server | Your plan or workspace does not allow it, or an administrator has not enabled it. Ask the workspace administrator. |
| You do not know whether the role took effect | Wait 30 minutes and check again. Do not keep creating tunnels; use the one you already have. |
| You are unsure which key is which | The runtime key authenticates the tunnel client. An admin key is more powerful and is not used here. |

## Done when

You have the tunnel ID, a runtime key in a safe place, the permissions, a workspace that allows custom connectors, and the tunnel associated with that workspace.

## Next

[Connect ChatGPT](06-connect-chatgpt.md)
