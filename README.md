# AI Coding Instructions & Best Practices

This repository hosts curated **"system instructions"** and **"rules"** for AI coding assistants (GitHub Copilot, Cursor, Windsurf, etc.) to ensure they generate high-quality, production-ready code.

## 🎯 Goal

AI models are powerful but often generate generic or outdated code. By providing them with specific constraints, best practices, and preferred patterns, we can force them to adhere to **production standards** (in this case, focusing on enterprise Spring Boot & Java).

## 📂 Available Stacks

### [Spring Boot 3.5 & Java 21](./spring-boot/)

Comprehensive rules for building modern enterprise applications.

- [**Main Instructions File**](./spring-boot/.github-copilot-instructions.md) (The "Master Prompt")
- **Deep Dive Rules:**
  - [Refining Transactions](./spring-boot/copilot-rules/transactions.md)
  - [Microservices Resilience](./spring-boot/copilot-rules/resilience.md)
  - [Modern Java Features](./spring-boot/copilot-rules/java-features.md)
  - [Security Best Practices](./spring-boot/copilot-rules/security.md)
  - ...and [many more](./spring-boot/README.md).

## 🚀 How to Use

### For GitHub Copilot

1. Copy the contents of the relevant `.github-copilot-instructions.md` into your own repository's `.github/copilot-instructions.md`.
2. (Optional) Copy the `copilot-rules/` folder to your repo to reference specific deep-dive context.

### For Cursor

1. Copy the `copilot-rules/` directory associated with your stack into your project's `.cursor/rules/` folder.
2. Cursor will index these files and use them to enforce project-specific logic.

## 🤝 Contributing

Contributions are welcome! If you have optimized prompts or rules for other stacks (React, Python, Go, etc.), please submit a Pull Request.
