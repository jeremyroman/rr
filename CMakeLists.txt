# *-* Mode: cmake; *-*

cmake_minimum_required(VERSION 2.8.5)
project(rr C CXX ASM)

enable_testing()
set(BUILD_SHARED_LIBS ON)

set(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)
set(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)

# CAREFUL!  "-" is an invalid character in RPM package names, while
# debian is happy with it.  However, "_" is illegal in debs, while RPM
# is cool with it.  Sigh.
set(rr_VERSION_MAJOR 4)
set(rr_VERSION_MINOR 0)
set(rr_VERSION_PATCH 2)

add_definitions(-DRR_VERSION="${rr_VERSION_MAJOR}.${rr_VERSION_MINOR}.${rr_VERSION_PATCH}")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pthread -O0 -g3 -Wall -Werror -Wstrict-prototypes")
# Define __STDC_LIMIT_MACROS so |#include <stdint.h>| works as expected.
# Define __STDC_FORMAT_MACROS so |#include <inttypes.h>| works as expected.
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D__USE_LARGEFILE64 -D__STDC_LIMIT_MACROS -D__STDC_FORMAT_MACROS -std=c++0x -pthread -O0 -g3 -Wall -Werror")
set(CMAKE_ASM_FLAGS "${CMAKE_ASM_FLAGS} -g3")

# Check that a 32-bit cross-compile works. This is needed regardless
# of whether the entire build is being built 32-bit.
if(CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "x86_64")
  # try_compile won't accept LINK_FLAGS, so do this manually.
  file(WRITE "${CMAKE_BINARY_DIR}/test32.c" "int main() { return 0; }")
  execute_process(COMMAND ${CMAKE_CXX_COMPILER} -o ${CMAKE_BINARY_DIR}/test32 ${CMAKE_BINARY_DIR}/test32.c -m32
			RESULT_VARIABLE COMPILER_32BIT_RESULT)
  if(NOT (COMPILER_32BIT_RESULT EQUAL 0))
    message(FATAL_ERROR "Your toolchain doesn't support 32-bit cross-compilation.")
  endif()
endif()

option(force32bit, "Force a 32-bit build, rather than a 64-bit one")
if(force32bit)
  set(rr_64BIT false)
  set(rr_MBITNESS_OPTION -m32)
else()
  if (CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(rr_64BIT true)
  else()
    set(rr_64BIT false)
  endif()
  set(rr_MBITNESS_OPTION)
endif()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${rr_MBITNESS_OPTION}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${rr_MBITNESS_OPTION}")
set(CMAKE_ASM_FLAGS "${CMAKE_ASM_FLAGS} ${rr_MBITNESS_OPTION}")

find_package(PkgConfig REQUIRED)

# If we're cross-compiling a 32-bit build on a 64-bit host we need
# to ensure we're looking for the right libraries.
# This has been tested on Ubuntu and Fedora.
set(LIBDIR32_CANDIDATES
  /usr/lib/i386-linux-gnu/pkgconfig/
  /usr/lib/pkgconfig/
)
if((NOT rr_64BIT) AND CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "x86_64")
  foreach(libdir ${LIBDIR32_CANDIDATES})
    if(IS_DIRECTORY ${libdir})
      set(ENV{PKG_CONFIG_LIBDIR} ${libdir})
      break()
     endif()
   endforeach(libdir)
   if (NOT DEFINED ENV{PKG_CONFIG_LIBDIR})
     message(FATAL_ERROR "Couldn't find a suitable 32-bit pkgconfig lib dir. You probably need to install a 32-bit pkgconfig package (pkgconfig.i686 for Fedora or pkg-config:i386 for Ubuntu")
   endif()
endif()

# Check for required libraries
set(REQUIRED_LIBS
  zlib
)
foreach(required_lib ${REQUIRED_LIBS})
  string(TOUPPER ${required_lib} PKG)
  pkg_check_modules(${PKG} REQUIRED ${required_lib})
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${${PKG}_CFLAGS}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${${PKG}_CFLAGS}")
endforeach(required_lib)

# Check for Python >=2.7 but not Python 3.
find_package(PythonInterp 2.7 REQUIRED)
if(PYTHON_VERSION_MAJOR GREATER 2)
  message(FATAL_ERROR "Python 3 is not supported, please use Python 2.7.")
endif()

# Check for required Python modules
set(REQUIRED_PYTHON_MODULES
  pexpect
)
foreach(py_module ${REQUIRED_PYTHON_MODULES})
  execute_process(COMMAND "${PYTHON_EXECUTABLE}" "-c"
			"import ${py_module}"
			RESULT_VARIABLE module_status)
  if(module_status)
    message(FATAL_ERROR "Couldn't find required Python module ${py_module}.")
  endif()
endforeach(py_module)

set_source_files_properties(src/preload/preload.c PROPERTIES COMPILE_FLAGS -O2)

include_directories("${PROJECT_SOURCE_DIR}/include")
# We need to know where our generated files are.
include_directories("${CMAKE_CURRENT_BINARY_DIR}")

add_library(rrpreload
  src/preload/preload.c
  src/preload/raw_syscall.S
  src/preload/syscall_hook.S
  src/preload/breakpoint_table.S
)

# Ensure that CMake knows about our generated files.
#
# Alphabetical, please.
set(GENERATED_FILES
  AssemblyTemplates.generated
  CheckSyscallNumbers.generated
  SyscallEnumsX64.generated
  SyscallEnumsX86.generated
  SyscallEnumsForTestsX64.generated
  SyscallEnumsForTestsX86.generated
  SyscallHelperFunctions.generated
  SyscallnameArch.generated
  SyscallRecordCase.generated
)

foreach(generated_file ${GENERATED_FILES})
  set_source_files_properties(${generated_file}
                              PROPERTIES GENERATED true HEADER_FILE_ONLY true)
  add_custom_command(OUTPUT ${generated_file}
                     COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/src/generate_syscalls.py"
		               "${CMAKE_CURRENT_BINARY_DIR}/${generated_file}"
		     DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/generate_syscalls.py"
		       "${CMAKE_CURRENT_SOURCE_DIR}/src/syscalls.py"
		       "${CMAKE_CURRENT_SOURCE_DIR}/src/assembly_templates.py")
endforeach(generated_file)

add_custom_target(Generated DEPENDS ${GENERATED_FILES})

add_custom_command(OUTPUT rr_page_64
                   COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/src/generate_rr_page.py"
                   "${CMAKE_CURRENT_BINARY_DIR}/bin/rr_page_64"
                   DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/generate_rr_page.py")
add_custom_command(OUTPUT rr_page_32
                   COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/src/generate_rr_page.py"
                   "${CMAKE_CURRENT_BINARY_DIR}/bin/rr_page_32"
                   DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/generate_rr_page.py")
add_custom_command(OUTPUT rr_page_64_replay
                   COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/src/generate_rr_page.py"
                   "${CMAKE_CURRENT_BINARY_DIR}/bin/rr_page_64_replay"
                   DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/generate_rr_page.py")
add_custom_command(OUTPUT rr_page_32_replay
                   COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/src/generate_rr_page.py"
                   "${CMAKE_CURRENT_BINARY_DIR}/bin/rr_page_32_replay"
                   DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/generate_rr_page.py")

add_custom_target(Pages DEPENDS rr_page_32 rr_page_64 rr_page_32_replay rr_page_64_replay)

add_executable(rr
  src/test/cpuid_loop.S
  src/AddressSpace.cc
  src/AutoRemoteSyscalls.cc
  src/Command.cc
  src/CompressedReader.cc
  src/CompressedWriter.cc
  src/CPUIDBugDetector.cc
  src/DiversionSession.cc
  src/DumpCommand.cc
  src/EmuFs.cc
  src/Event.cc
  src/ExtraRegisters.cc
  src/fast_forward.cc
  src/FdTable.cc
  src/Flags.cc
  src/GdbConnection.cc
  src/GdbExpression.cc
  src/GdbInitCommand.cc
  src/GdbServer.cc
  src/HelpCommand.cc
  src/kernel_abi.cc
  src/kernel_metadata.cc
  src/log.cc
  src/MagicSaveDataMonitor.cc
  src/main.cc
  src/Monkeypatcher.cc
  src/PerfCounters.cc
  src/PsCommand.cc
  src/RecordCommand.cc
  src/RecordSession.cc
  src/record_signal.cc
  src/record_syscall.cc
  src/Registers.cc
  src/remote_code_ptr.cc
  src/ReplayCommand.cc
  src/ReplaySession.cc
  src/replay_syscall.cc
  src/ReplayTimeline.cc
  src/Scheduler.cc
  src/SeccompFilterRewriter.cc
  src/Session.cc
  src/StdioMonitor.cc
  src/task.cc
  src/TraceFrame.cc
  src/TraceStream.cc
  src/util.cc
)
add_dependencies(rr Generated Pages)

target_link_libraries(rr
  -ldl
  -lrt
  ${ZLIB_LDFLAGS}
)

target_link_libraries(rrpreload
  -ldl
)

add_executable(exec_stub src/exec_stub.c)
set_target_properties(exec_stub PROPERTIES LINK_FLAGS -nostdlib)
set_source_files_properties(src/exec_stub.c
                            COMPILE_FLAGS "-fno-stack-protector")

install(PROGRAMS scripts/signal-rr-recording.sh
                  ${CMAKE_CURRENT_BINARY_DIR}/bin/rr_page_64
                  ${CMAKE_CURRENT_BINARY_DIR}/bin/rr_page_64_replay
                  ${CMAKE_CURRENT_BINARY_DIR}/bin/rr_page_32
                  ${CMAKE_CURRENT_BINARY_DIR}/bin/rr_page_32_replay
  DESTINATION bin)

install(TARGETS rr rrpreload exec_stub
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib)

# Build 32-bit librrpreload on 64-bit builds.
# We copy the source files into '32' subdirectories in the output
# directory, so we can set different compile options on them.
# This sucks but I can't find a better way to get CMake to build
# the same source file in two different ways.
if(rr_64BIT)
  foreach(file preload_interface.h)
    set_source_files_properties(32/${file}
                                PROPERTIES GENERATED true HEADER_FILE_ONLY true)
    add_custom_command(OUTPUT 32/${file}
                       COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/32 &&
                               cp "${CMAKE_CURRENT_SOURCE_DIR}/src/preload/${file}"
                                  "${CMAKE_CURRENT_BINARY_DIR}/32/${file}"
	               DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/preload/${file}")
  endforeach(file)

  foreach(file preload.c raw_syscall.S syscall_hook.S breakpoint_table.S)
    set_source_files_properties(32/${file}
                                PROPERTIES GENERATED true COMPILE_FLAGS "-m32 -O2")
    add_custom_command(OUTPUT 32/${file}
                       COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/32 &&
                               cp "${CMAKE_CURRENT_SOURCE_DIR}/src/preload/${file}"
                                  "${CMAKE_CURRENT_BINARY_DIR}/32/${file}"
	               DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/preload/${file}"
                               32/preload_interface.h)
  endforeach(file)

  add_library(rrpreload_32
    32/preload.c
    32/raw_syscall.S
    32/syscall_hook.S
    32/breakpoint_table.S
  )
  set_target_properties(rrpreload_32 PROPERTIES LINK_FLAGS -m32)
  target_link_libraries(rrpreload_32
    -ldl
  )

  foreach(file exec_stub.c)
    set_source_files_properties(32/${file}
                                PROPERTIES GENERATED true COMPILE_FLAGS "-m32 -fno-stack-protector")
    add_custom_command(OUTPUT 32/${file}
                       COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/32 &&
                               cp "${CMAKE_CURRENT_SOURCE_DIR}/src/${file}"
                                  "${CMAKE_CURRENT_BINARY_DIR}/32/${file}")
  endforeach(file)

  add_executable(exec_stub_32 32/exec_stub.c)
  set_target_properties(exec_stub_32 PROPERTIES LINK_FLAGS "-nostdlib -m32")

  install(TARGETS rrpreload_32 exec_stub_32
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib)
endif()

##--------------------------------------------------
## Testing

# A "basic test" consists of a foo.c source file. All basic tests use the
# same basic_test.run driver script. The test name is passed as an additional
# parameter to the driver script. This script just does
# "compare_test EXIT-SUCCESS", i.e. records and replays the program and verifies
# that the output of both runs is identical and contains EXIT-SUCCESS.
#
# NB: you must update this variable when adding a new test source
# file.  The list is not generated automatically.
#
# Alphabetical, please.
set(BASIC_TESTS
  64bit_child
  accept
  alarm
  alarm2
  alsa_ioctl
  arch_prctl
  async_segv_ignored
  async_signal_syscalls2
  at_threadexit
  bad_ip
  bad_syscall
  barrier
  big_buffers
  block
  blocked_sigsegv
  brk
  brk2
  capget
  chew_cpu
  chown
  clock
  clone
  clone_bad_stack
  clone_immediate_exit
  clone_untraced
  constructor
  desched_blocking_poll
  dup
  epoll_create
  epoll_create1
  exec_self
  fadvise
  fault_in_code_page
  fcntl_owner_ex
  fcntl_dupfd
  fcntl_seals
  fcntl_sig
  fd_tracking_across_threads
  fds_clean
  flock
  flock2
  fork_brk
  fork_child_crash
  fork_stress
  fxregs
  getgroups
  getrandom
  setitimer
  getsid
  gettimeofday
  grandchild_threads
  grandchild_threads_main_running
  grandchild_threads_thread_running
  grandchild_threads_parent_alive
  ignored_sigsegv
  int3
  intr_futex_wait_restart
  intr_poll
  intr_pselect
  intr_read_no_restart
  intr_read_restart
  intr_sleep
  intr_sleep_no_restart
  invalid_fcntl
  io
  ioctl
  legacy_ugid
  madvise
  map_fixed
  memfd_create
  mincore
  mknod
  mlock
  mmap_discontinuous
  mmap_private
  mmap_ro
  mmap_shared
  mmap_shared_multiple
  mmap_shared_subpage
  mmap_short_file
  mmap_tmpfs
  mprotect
  mprotect_growsdown
  mprotect_heterogenous
  mprotect_none
  mprotect_stack
  mremap
  mremap_shrink
  msg
  msync
  multiple_pending_signals
  multiple_pending_signals_sequential
  munmap_segv
  munmap_discontinuous
  no_mask_timeslice
  numa
  old_fork
  orphan_process
  pause
  perf_event
  personality
  pthread_rwlocks
  poll_sig_race
  prctl
  prctl_deathsig
  prctl_name
  protect_rr_fds
  prw
  pthread_condvar_locking
  ptrace
  ptrace_attach_null_status
  ptrace_attach_running
  ptrace_attach_sleeping
  ptrace_attach_stopped
  ptrace_attach_thread_running
  ptrace_signals
  ptracer_death
  ptracer_death_multithread
  ptracer_death_multithread_peer
  quotactl
  rdtsc
  read_nothing
  readdir
  readlink
  readlinkat
  readv
  rlimit
  robust_futex
  rusage
  save_data_fd
  sched_setaffinity
  sched_setparam
  sched_yield
  sched_yield_to_lower_priority
  scm_rights
  seccomp
  seccomp_null
  self_sigint
  sem
  sendfile
  set_ptracer
  set_tid_address
  setgid
  setgroups
  setsid
  setuid
  shm
  sigaction_old
  sigaltstack
  sigchld_interrupt_signal
  sighandler_fork
  sigill
  signalfd
  sigprocmask
  sigprocmask_in_syscallbuf_sighandler
  sigprocmask_syscallbuf
  sigqueueinfo
  sigreturn
  sigreturn_reg
  sigrt
  sigstop
  sigstop2
  sigsuspend
  sigtrap
  simple
  sioc
  sock_names_opts
  splice
  stack_overflow
  stack_overflow_altstack
  stack_overflow_with_guard
  statfs
  stdout_child
  stdout_cloexec
  stdout_dup
  stdout_redirect
  strict_priorities
  switch_read
  sync
  syscallbuf_signal_reset
  syscallbuf_timeslice
  syscallbuf_timeslice2
  sysconf
  sysctl
  sysemu_singlestep
  sysinfo
  tcgets
  tgkill
  thread_stress
  thread_yield
  timer
  timerfd
  times
  tiocgwinsz
  truncate
  tty_ioctls
  uname
  unjoined_thread
  unshare
  utimes
  vfork_flush
  video_capture
  wait
  write_race
  writev
  xattr
  zero_length_read
)

# A "test with program" consists of a foo.c source file and a foo.run driver
# script.  See src/test/util.sh to learn how the .run files work.
#
# NB: you must update this variable when adding a new test source
# file.  The list is not generated automatically.
#
# Alphabetical, please.
set(TESTS_WITH_PROGRAM
  abort_nonmain
  args
  async_kill_with_threads
  async_kill_with_threads_main_running
  async_kill_with_threads_thread_running
  async_segv
  async_signal_syscalls
  async_signal_syscalls_siginfo
  async_usr1
  block_intr_sigchld
  blocked_bad_ip
  breakpoint
  breakpoint_conditions
  breakpoint_overlap
  call_function
  checkpoint_dying_threads
  checkpoint_mixed_mode
  clone_interruption
  clone_vfork
  conditional_breakpoint_calls
  conditional_breakpoint_offload
  condvar_stress
  crash
  crash_in_function
  exit_group
  exit_status
  explicit_checkpoints
  fork_syscalls
  function_calls
  getcwd
  goto_event
  hello
  ignored_async_usr1
  immediate_restart
  interrupt
  intr_ptrace_decline
  link
  madvise_dontfork
  main_thread_exit
  mmap_shared_prot
  mmap_write
  mutex_pi_stress
  nanosleep
  priority
  read_big_struct
  restart_abnormal_exit
  reverse_continue_breakpoint
  reverse_continue_multiprocess
  reverse_continue_process_signal
  reverse_step_long
  reverse_step_threads
  reverse_step_threads_break
  segfault
  shared_persistent_file
  signal_numbers
  stack_growth
  step_thread
  string_instructions
  string_instructions_replay
  string_instructions_watch
  syscallbuf_fd_disabling
  target_fork
  target_process
  term_nonmain
  threaded_syscall_spam
  threads
  tiocinq
  unexpected_stack_growth
  user_ignore_sig
  vfork
  watchpoint
  watchpoint_syscall
  watchpoint_unaligned
)

# A "test without program" is a foo.run driver script only, which does
# something with one of the test executables above (or has special rules
# to build its own executable).
#
# NB: you must update this variable when adding a new test source
# file.  The list is not generated automatically.
#
# Alphabetical, please.
set(TESTS_WITHOUT_PROGRAM
  async_signal_syscalls_100
  async_signal_syscalls_1000
  bad_breakpoint
  break_block
  break_clock
  break_clone
  break_exec
  break_int3
  break_mmap_private
  break_msg
  break_rdtsc
  break_sigreturn
  break_sync_signal
  break_thread
  break_time_slice
  breakpoint_consistent
  call_exit
  check_patched_pthread
  checkpoint_async_signal_syscalls_1000
  checkpoint_mmap_shared
  checkpoint_prctl_name
  checkpoint_simple
  cont_signal
  cpuid
  dead_thread_target
  desched_ticks
  deliver_async_signal_during_syscalls
  env_newline
  exec_stop
  execp
  explicit_checkpoint_clone
  final_sigkill
  first_instruction
  fork_exec_info_thr
  get_thread_list
  hardlink_mmapped_files
  parent_no_break_child_bkpt
  parent_no_stop_child_crash
  read_bad_mem
  remove_watchpoint
  restart_invalid_checkpoint
  restart_unstable
  restart_diversion
  reverse_alarm
  reverse_continue_exec_subprocess
  reverse_continue_fork_subprocess
  reverse_continue_start
  reverse_finish
  reverse_step_breakpoint
  reverse_step_signal
  reverse_step_threads2
  reverse_watchpoint
  reverse_watchpoint_syscall
  run_in_function
  sanity
  shm_checkpoint
  signal_stop
  signal_checkpoint
  simple_script
  simple_script_debug
  simple_winch
  stack_overflow_debug
  step1
  step_rdtsc
  step_signal
  string_instructions_break
  string_instructions_replay_quirk
  subprocess_exit_ends_session
  switch_processes
  syscallbuf_timeslice_250
  trace_version
  term_trace_cpu
  term_trace_syscall
  when
)

foreach(test ${BASIC_TESTS} ${TESTS_WITH_PROGRAM})
  add_executable(${test} src/test/${test}.c)
  add_dependencies(${test} Generated)
  target_link_libraries(${test} -lrt)
endforeach(test)

add_library(test_lib
  src/test/test_lib.c
)
add_dependencies(test_lib Generated)
target_link_libraries(constructor -lrt test_lib)

# cpuid test needs to link with cpuid_loop.S
add_executable(cpuid src/test/cpuid.c src/test/cpuid_loop.S)
add_dependencies(cpuid Generated)
target_link_libraries(cpuid -lrt)

foreach(test ${BASIC_TESTS} ${OTHER_TESTS})
  add_test(${test}
    bash ${CMAKE_SOURCE_DIR}/src/test/basic_test.run -b ${CMAKE_SOURCE_DIR} ${PROJECT_BINARY_DIR} ${test})
  set_tests_properties(${test}
    PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")

  add_test(${test}-no-syscallbuf
    bash ${CMAKE_SOURCE_DIR}/src/test/basic_test.run -n ${CMAKE_SOURCE_DIR} ${PROJECT_BINARY_DIR} ${test})
  set_tests_properties(${test}-no-syscallbuf
    PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")
endforeach(test)

foreach(test ${TESTS_WITH_PROGRAM} ${TESTS_WITHOUT_PROGRAM})
  add_test(${test}
    bash ${CMAKE_SOURCE_DIR}/src/test/${test}.run -b ${CMAKE_SOURCE_DIR} ${PROJECT_BINARY_DIR} ${test})
  set_tests_properties(${test}
    PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")

  add_test(${test}-no-syscallbuf
    bash ${CMAKE_SOURCE_DIR}/src/test/${test}.run -n ${CMAKE_SOURCE_DIR} ${PROJECT_BINARY_DIR} ${test})
  set_tests_properties(${test}-no-syscallbuf
    PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")
endforeach(test)

# Run 32-bit tests on 64-bit builds.
# We copy the test files into '32' subdirectories in the output
# directory, so we can set different compile options on them.
# This sucks but I can't find a better way to get CMake to build
# the same source file in two different ways.
if(rr_64BIT)
  set_source_files_properties(32/rrutil.h
                              PROPERTIES GENERATED true HEADER_FILE_ONLY true)
  add_custom_command(OUTPUT 32/rrutil.h
                     COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/32 &&
                             mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/bin/32 &&
                             cp -f "${CMAKE_CURRENT_SOURCE_DIR}/src/test/rrutil.h"
                                "${CMAKE_CURRENT_BINARY_DIR}/32/rrutil.h"
  	             DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/test/rrutil.h")

  foreach(test ${BASIC_TESTS} ${TESTS_WITH_PROGRAM} cpuid test_lib)
    set_source_files_properties(32/${test}.c
                                PROPERTIES GENERATED true COMPILE_FLAGS -m32)
    add_custom_command(OUTPUT 32/${test}.c
                       COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/32 &&
                               cp "${CMAKE_CURRENT_SOURCE_DIR}/src/test/${test}.c"
                                  "${CMAKE_CURRENT_BINARY_DIR}/32/${test}.c"
	               DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/test/${test}.c" 32/rrutil.h)
  endforeach(test)

  foreach(file cpuid_loop.S)
    set_source_files_properties(32/${file}
                                PROPERTIES GENERATED true COMPILE_FLAGS -m32)
    add_custom_command(OUTPUT 32/${file}
                       COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/32 &&
                               cp "${CMAKE_CURRENT_SOURCE_DIR}/src/test/${file}"
                                  "${CMAKE_CURRENT_BINARY_DIR}/32/${file}"
	               DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/test/${file}")
  endforeach(file)

  foreach(test ${BASIC_TESTS} ${TESTS_WITH_PROGRAM})
    add_executable(${test}_32 32/${test}.c)
    add_dependencies(${test}_32 Generated)
    set_target_properties(${test}_32 PROPERTIES LINK_FLAGS -m32)
    target_link_libraries(${test}_32 -lrt)
  endforeach(test)

  add_library(test_lib_32
    32/test_lib.c
  )
  add_dependencies(test_lib_32 Generated)
  set_target_properties(test_lib_32 PROPERTIES LINK_FLAGS -m32)

  target_link_libraries(constructor_32 -lrt test_lib_32)

  # cpuid test needs to link with cpuid_loop.S
  add_executable(cpuid_32 32/cpuid.c 32/cpuid_loop.S)
  add_dependencies(cpuid_32 Generated)
  set_target_properties(cpuid_32 PROPERTIES LINK_FLAGS -m32)
  target_link_libraries(cpuid_32 -lrt)

  foreach(test ${BASIC_TESTS} ${OTHER_TESTS})
    add_test(${test}-32
      bash ${CMAKE_SOURCE_DIR}/src/test/basic_test.run -b ${CMAKE_SOURCE_DIR} ${PROJECT_BINARY_DIR} ${test}_32)
    set_tests_properties(${test}-32
      PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")

    add_test(${test}-32-no-syscallbuf
      bash ${CMAKE_SOURCE_DIR}/src/test/basic_test.run -n ${CMAKE_SOURCE_DIR} ${PROJECT_BINARY_DIR} ${test}_32)
    set_tests_properties(${test}-32-no-syscallbuf
      PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")
  endforeach(test)

  foreach(test ${TESTS_WITH_PROGRAM} ${TESTS_WITHOUT_PROGRAM})
    add_test(${test}-32
      bash ${CMAKE_SOURCE_DIR}/src/test/${test}.run -b ${CMAKE_SOURCE_DIR} ${PROJECT_BINARY_DIR} ${test}_32)
    set_tests_properties(${test}-32
      PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")

    add_test(${test}-32-no-syscallbuf
      bash ${CMAKE_SOURCE_DIR}/src/test/${test}.run -n ${CMAKE_SOURCE_DIR} ${PROJECT_BINARY_DIR} ${test}_32)
    set_tests_properties(${test}-32-no-syscallbuf
      PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")
  endforeach(test)
endif()

include(ProcessorCount)
ProcessorCount(N)
if(NOT N EQUAL 0)
  set(JFLAG -j${N})
endif()

add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND} --verbose ${JFLAG})
# Run only syscallbuf-enabled and native-bitness tests
add_custom_target(fastcheck COMMAND ${CMAKE_CTEST_COMMAND} --verbose --exclude-regex '[-]' ${JFLAG})

##--------------------------------------------------
## Package configuration

include (InstallRequiredSystemLibraries)

set(CPACK_PACKAGE_NAME "rr")
set(CPACK_PACKAGE_VERSION_MAJOR "${rr_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${rr_VERSION_MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${rr_VERSION_PATCH}")
set(CPACK_SYSTEM_NAME "${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}")

set(CPACK_OUTPUT_FILE_PREFIX dist)
set(CPACK_GENERATOR "TGZ;RPM;DEB")
set(CPACK_SOURCE_GENERATOR "TGZ")
set(CPACK_BINARY_DIR "${PROJECT_BINARY_DIR}")
set(CPACK_STRIP_FILES TRUE)

set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_SOURCE_DIR}/LICENSE")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY
  "Lightweight tool for recording and replaying execution of applications (trees of processes and threads)")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_SOURCE_DIR}/README.md")
set(CPACK_PACKAGE_VENDOR "Mozilla Foundation")

set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Mozilla Foundation")
set(CPACK_DEBIAN_PACKAGE_SECTION "devel")
if(${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86_64")
  set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "amd64")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "i.86")
  set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "i386")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm.*")
  set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "arm")
endif()

# XXX Cmake 2.8.7 doesn't know how to avoid specifiying /usr,
# /usr/bin, etc, as files to be installed, but distros are finicky
# about their specification.  We want to manually filter those paths
# out of our install list but 2.8.7 also isn't capable of that.
set(CPACK_RPM_USER_BINARY_SPECFILE "${CMAKE_SOURCE_DIR}/rr.spec")
set(CPACK_RPM_PACKAGE_RELEASE 1)
set(CPACK_RPM_PACKAGE_GROUP "Development/Debuggers")
set(CPACK_RPM_PACKAGE_LICENSE "MIT and BSD")

include (CPack)

##--------------------------------------------------
## Misc

add_custom_target(setup-travis COMMAND src/script/setup_travis.sh)
