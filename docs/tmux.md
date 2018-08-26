## .tmux.conf
~~~
# .tmux.conf
set -g default-terminal screen-256color

# Enable utf8
#set-window-option -g utf8 on

# Set prefix
set -g prefix C-a
unbind C-a
bind C-a send-prefix

# begin with 1
set -g base-index 1

# Automatically set window title
set-window-option -g automatic-rename on
set-option -g set-titles on


# Change history limit from the default 2000
set-option -g history-limit 10000

# Enable the live reload of tmux configuration file
unbind r
bind r source-file ~/.tmux.conf

# screen split
bind - split-window -v
bind _ split-window -h

# vim like split screen navigation
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# switch panes using Alt-arrow without prefix
bind -n M-Left select-pane -L
bind -n M-Right select-pane -R
bind -n M-Up select-pane -U
bind -n M-Down select-pane -D


# Shift arrow to switch windows
bind -n S-Left  previous-window
bind -n S-Right next-window

# Mouse
# set -g mode-mouse on
# set-option -g mouse-select-pane on
# set-option -g mouse-select-window on
# set-option -g mouse-resize-pane on
# set -g mouse



###---------------------###
### Colors              ###
###---------------------###

# Colorize messages in the command line
set-option -g message-bg black
set-option -g message-fg brightred

###---------------------###
### Status bar          ###
###---------------------###
# Turn on status bar
set-option -g status on

# Enable utf8 for status bar
#set -g status-utf8 on

# Set the status bar update frequency
set -g status-interval 5

# Position the status bar
set -g status-justify centre

# Visual notification of activity in other windows
setw -g monitor-activity off
set -g visual-activity off

# Set color for status bar
set-option -g status-bg colour235
set-option -g status-fg yellow
set-option -g status-attr bright

# Set window list colors: active - red, inactive - cyan
set-window-option -g window-status-fg brightblue
set-window-option -g window-status-bg colour236
set-window-option -g window-status-attr dim

set-window-option -g window-status-current-fg yellow
#set-window-option -g window-status-current-bg colour235
set-window-option -g window-status-current-bg colour124
# set-window-option -g window-status-current-attr bright
set-window-option -g window-status-current-attr dim


# border colours
set -g pane-border-fg magenta
set -g pane-active-border-fg green
set -g pane-active-border-bg default
#
# Show host name and IP address on left side of status bar
set -g status-left-length 70
set -g status-left '#[fg=blue,bold][#S]#[default]'

# Show session name, window & pane number, dae and time on right side of status bar
set -g status-right-length 60
set -g status-right '|#[fg=magenta,bold]#(whoami)#[default]| #[fg=blue,bold][%a %d/%m %H:%M]#[default]'

# Vi copypaste mode
set-window-option -g mode-keys vi
bind-key -t vi-copy 'v' begin-selection
bind-key -t vi-copy 'y' copy-selection


# should be the last line
#run-shell ~/tmux-resurrect/resurrect.tmux
# colors
#source-file ~/.tmux.colors
#\; display "Reloaded ~/.tmux.conf"
~~
