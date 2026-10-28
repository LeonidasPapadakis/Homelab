# Remote access with Tailscale

Nothing listens on the internet, all remote access rides the Tailnet.

## Subnet routers

| Router | Advertises | Users |
| --- | --- | --- |
| admin-portal | The whole `/16` Powerhouse site | Devices on my own private tailnet |
| family-portal | Services VLAN + individual agent back-ends | Separate tailnet for family and friends |

## Access list

My Tailscale ACL uses:
- groups for people (admins, family); rules name groups, not accounts, so adding a person doesn't touch the policy
- hosts map so rules use a hostname instead of an IP

```json
{
	// ---------------------------------------------------------------
	// Groups
	// ---------------------------------------------------------------
	"groups": {
		"group:admins": ["admin@example.com"],
		"group:family": [
			"brother@example.com",
			"sister@example.com",
		],
	},

	// ---------------------------------------------------------------
	// LAN machines behind the subnet router
	// ---------------------------------------------------------------
	"hosts": {
		"service1":   "10.1.20.10",
		"agent1": "10.1.40.10",
	},

	// ---------------------------------------------------------------
	// Access rules: default deny, anything not listed here is blocked
	// ---------------------------------------------------------------
	"grants": [
		// Admin: full access to everything
		{
			"src": ["group:admins"],
			"dst": ["*"],
			"ip":  ["*"],
		},

		// Family group: access to services
		{
			"src": ["group:family"],
			"dst": ["service1"],
			"ip":  ["tcp:3000"],
		},

		// Individual users: access to specific machines
		{
			"src": ["brother@example.com"],
			"dst": ["agent1"],
			"ip":  ["*"],
		},
	],

	// ---------------------------------------------------------------
	// Tests: the policy is rejected on save if any of these fail
	// ---------------------------------------------------------------
	"tests": [
		{
			"src":    "admin@example.com",
			"accept":   ["10.1.10.10:8080"],
		},
    {
			"src":    "brother@example.com",
			"accept": ["10.1.20.10:3000"],
			"deny":   ["10.1.10.10:8080"],
		},
	],
}

```