---
layout: post
title: "Testando IPC entre GTA SA e Elixir"
date: 2025-05-12 23:23:40 0000
categories: C++ Hacking Elixir Erlang
---

Olá mundo!

Essa semana comecei a aprimorar a maneira como estava testando as funcionalidades e descobertas do meu processo de engenharia reversa com o GTA SA.

Infelizmente o processo ainda estava bem manual, onde eu precisava estar constantemente repetindo os seguintes passos:

1. Engenharia Reversa (IDA Pro) até descobrir coisas que poderiam ser úteis.
2. Reescrita de código seja de função ou pra conseguir acessar estar possíveis informações in-game pela minha DLL
3. Injeção da DLL e então conseguir rodar os testes...

A partir dessa dificuldade em conseguir iterar mais rápido, me surgiu a seguinte ideia: criar um terceiro elemento que não vai precisar ser injetado (DLL), e pode executar atividades mais complexas e dinâmicas.

Assim, meio que pivotando a ideia da minha biblioteca, servir apenas como uma casca de leituras/escritas de memória privilegiada dentro do processo, tendo em vista que a memória virtual do mesmo vai estar 100% disponível.

A primeira versão desse meu teste foi estabelecer uma comunicação entre processos utilizando o conceito de NamedPipes [(IPC)](https://en.wikipedia.org/wiki/Inter-process_communication).

Como isso funciona na minha biblioteca:
```
void HandleTeleport(std::istringstream& args) {
    Vector3 pos;
    if (args >> pos.x >> pos.y >> pos.z) {
        DWORD* playerPtr = GetPlayer();
        CPed player((uintptr_t)playerPtr);

        printf("you id: %p\n", playerPtr);

        player.Teleport(pos, 0);
        printf("[TP] Teleporting to: %f, %f, %f\n", pos.x, pos.y, pos.z);
    }
    else {
        printf("[TP] Invalid vector input.\n");
    }
}

void HandleCreateVeh(std::istringstream& args) {
    int modelId;

    if (args >> modelId) {
        int response = SpawnVehicle(modelId);
        printf("[Create Vehicle] model: %d\n", modelId);
    }
    else {
        printf("[Create Vehicle] Invalid input\n");
    }
}

void DispatchCommand(const std::string& input) {
    std::istringstream iss(input);
    std::string cmd;
    iss >> cmd;

    typedef std::function<void(std::istringstream&)> CommandHandler;

    static const std::unordered_map<std::string, CommandHandler> commandMap = {
        { "tp", HandleTeleport },
        { "car", HandleCreateVeh }
    };

    for (std::unordered_map<std::string, CommandHandler>::const_iterator it = commandMap.begin(); it != commandMap.end(); ++it) {
        if (EqualsIgnoreCase(cmd, it->first)) {
            it->second(iss);
            return;
        }
    }

    printf("[Error] Unknown command: %s\n", cmd);
  
}

void NamedPipeThread() {
    const char* pipeName = R"(\\.\pipe\game_pipe)";

    // Define security attributes
    SECURITY_ATTRIBUTES sa;
    sa.nLength = sizeof(sa);
    sa.bInheritHandle = FALSE;

    // Initialize the security descriptor
    SECURITY_DESCRIPTOR sd;
    if (!InitializeSecurityDescriptor(&sd, SECURITY_DESCRIPTOR_REVISION)) {
        std::cerr << "Failed to initialize security descriptor. Error: " << GetLastError() << std::endl;
        return;
    }

    // Set a NULL DACL, which grants full access to everyone
    if (!SetSecurityDescriptorDacl(&sd, TRUE, NULL, FALSE)) {
        std::cerr << "Failed to set NULL DACL. Error: " << GetLastError() << std::endl;
        return;
    }

    // Assign the security descriptor to the SECURITY_ATTRIBUTES
    sa.lpSecurityDescriptor = &sd;

    while (true) {
        HANDLE hPipe = CreateNamedPipeA(
            pipeName,
            PIPE_ACCESS_DUPLEX,
            PIPE_TYPE_MESSAGE | PIPE_READMODE_MESSAGE | PIPE_WAIT,
            1,
            40,
            40,
            0,
            &sa
        );

        if (hPipe == INVALID_HANDLE_VALUE) {
            printf("Unable to create named pipe, check for %d!\n", GetLastError());
            return;
        }

        printf("Waiting for client connection...\n");
        if (!ConnectNamedPipe(hPipe, NULL)) {
            printf("Unable to connect into pipe, check for: %d!\n", GetLastError());
            CloseHandle(hPipe);
            return;
        }

        char buffer[64];
        DWORD bytesRead;

        while (ReadFile(hPipe, buffer, sizeof(buffer) - 1, &bytesRead, NULL)) {
            buffer[bytesRead] = '\0';
            printf("Message received: %s\n", buffer);
            DispatchCommand(buffer);
        }

        DisconnectNamedPipe(hPipe);
        CloseHandle(hPipe);
    }
}
```
Obviamente, no lado do jogo, a minha DLL despacha esta função como uma thread, para garantir que o mesmo vai ficar sendo escutado durante todo o processo de execução do jogo.

No lado do Elixir:
```
defmodule GameConnector do
  @pipe_name "\\\\.\\pipe\\game_pipe"

  def start_link() do
    # Start a process to manage the persistent file handle
    Task.start_link(fn -> communication_loop() end)
  end

  defp communication_loop() do
    with {:ok, file} <- File.open(@pipe_name, [:binary, :read, :write]) do
      try do
        # Continuous communication handling
        loop(file)
      after
        # Ensure the file is closed on exit
        File.close(file)
      end
    else
      {:error, err} ->
        IO.inspect("Unable to start your communication with GTA SA through msa.dll")
        IO.inspect(err)
    end
  end

  defp loop(file) do
    receive do
      {:send, message} ->
        IO.binwrite(file, message)
        loop(file)
    end
  end

  # API for sending a message
  def send_message(pid, message) do
    send(pid, {:send, message})
  end
end
```

E internamente também adicionei um AllocConsole pra ajudar no debug e revisão desse processo.