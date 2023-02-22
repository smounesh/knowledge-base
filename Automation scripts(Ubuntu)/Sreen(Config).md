## Screen #config

 1. By default screen will not show how many screen windows are running in a session

 2. This config will create a bottom bar which shows how may screen windows are running in a screen session along with the screen names

### Cmd:
```bash
cat ~/.screenrc
```
```yaml
# GNU Screen config file
# Filename: ~/.screenrc

startup_message off

vbell off
maxwin 8

defscrollback 5000

hardstatus on
hardstatus alwayslastline

termcapinfo xterm "ks=E[?1lE:kuE[A:kd=E[B:kl=E[D:kr=E[C:kh=E[5~:kH=E[F"
hardstatus alwayslastline "%{-b gk}%-w%{+b kg}%50>%n %t%{-b gk}%+w%< %= %{mk}%l %{ck}%d%{wk}-%{ck}%m %{gk}%c"
```

