defmodule SentinelMesh do
  use GenServer

  ## ===== Public API =====

  def start_link(name) do
    GenServer.start_link(__MODULE__, %{
      name: name,
      workers: %{},
      crash_log: %{}
    }, name: name)
  end

  def spawn_worker(manager, id, fun) do
    GenServer.call(manager, {:spawn, id, fun})
  end

  def kill_worker(manager, id) do
    GenServer.cast(manager, {:kill, id})
  end

  def audit(manager) do
    GenServer.call(manager, :audit)
  end

  ## ===== GenServer =====

  def init(state), do: {:ok, state}

  def handle_call({:spawn, id, fun}, _from, state) do
    pid = spawn_link(fn -> worker_loop(fun) end)

    Process.monitor(pid)

    workers = Map.put(state.workers, pid, id)
    {:reply, {:ok, pid}, %{state | workers: workers}}
  end

  def handle_call(:audit, _from, state) do
    {:reply, %{
      active_workers: map_size(state.workers),
      crash_log: state.crash_log
    }, state}
  end

  def handle_cast({:kill, id}, state) do
    pid =
      state.workers
      |> Enum.find(fn {_pid, wid} -> wid == id end)
      |> case do
        {pid, _} -> pid
        nil -> nil
      end

    if pid, do: Process.exit(pid, :kill)
    {:noreply, state}
  end

  def handle_info({:DOWN, _ref, :process, pid, reason}, state) do
    id = Map.get(state.workers, pid)

    crash_log =
      Map.update(state.crash_log, id, 1, &(&1 + 1))

    IO.puts("Worker #{id} crashed (#{inspect reason}), restarting...")

    new_pid = spawn_link(fn -> worker_loop(fn -> :ok end) end)
    Process.monitor(new_pid)

    workers =
      state.workers
      |> Map.delete(pid)
      |> Map.put(new_pid, id)

    {:noreply, %{state | workers: workers, crash_log: crash_log}}
  end

  ## ===== Worker =====

  defp worker_loop(fun) do
    fun.()
    receive do
      :crash -> exit(:simulated_failure)
    after
      1_000 ->
        worker_loop(fun)
    end
  end
end

## ===== Demo =====

defmodule Demo do
  def run do
    {:ok, mgr} = SentinelMesh.start_link(:mesh)

    SentinelMesh.spawn_worker(:mesh, :alpha, fn -> :ok end)
    SentinelMesh.spawn_worker(:mesh, :beta, fn -> :ok end)

    :timer.sleep(1000)
    SentinelMesh.kill_worker(:mesh, :alpha)

    :timer.sleep(2000)
    IO.inspect SentinelMesh.audit(:mesh)
  end
end

Demo.run()
