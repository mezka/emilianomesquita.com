---
layout: post
lang: en
title:  Navigate to a modal in AngularJS (with ui-router)
categories: ['AngularJS']
---

A while ago I found myself wanting to use `ui-router` to navigate to another view and open a modal, I enclosed this modal in the other view  within a directive element, whose content was shown or hidden through a `toggleModal` event.

{% highlight javascript %}
angular.module('app')
    .directive('exampleModal', function () {
    return {
        restrict: 'E',
        replace: true,
        template: ` <div class="modal" tabindex="-1" role="dialog">
                        <div class="modal-dialog modal-dialog-centered" role="document">
                            <div class="modal-content">
                            <div class="modal-header">
                                <h5 class="modal-title">Add new address</h5>
                                <button type="button" class="close" data-dismiss="modal" aria-label="Close">
                                <span aria-hidden="true">&times;</span>
                                </button>
                            </div>
                            <div class="modal-body">
                                <p>Hello</p>
                            </div>
                            <div class="modal-footer">
                                <button type="button" class="btn btn-secondary" data-dismiss="modal">Close</button>
                            </div>
                            </div>
                        </div>
                    </div> `,

        link: function(scope, element){

            scope.$on('toggleModal', function(){
                element.modal()
            });
            
            scope.toggle = function(){
                element.modal('toggle')
            };
        },
    }
});
{% endhighlight %}

At the time glossing over the state-level hooks, which only provided the hooks `onEnter`, `onExit`, and `onRetain`, I came up with this hack:

{% highlight javascript %}
const userControlPanelState = {
    name: 'usercp',
    url: '/user/cp',
    templateUrl: '/views/usercp/usercp.html',
    controller: 'userControlPanelController as userCP',
    data: { requiresAuth: true },
    params: { modal: false},
    onEnter: function($rootScope, $stateParams){
        if($stateParams.modal){
            setTimeout(() => { $rootScope.$broadcast('toggleModal') }, 0)
        }
    }
};
{% endhighlight %}

The idea is that I'm hooking to the `onEnter`, if the modal parameter is set to true I will broadcast a `toggleModal` event which the `exampleModal` directive will listen on and show the modal.

There are some caveats to this approach, if I call `$rootScope.$broadcast` here the state transition wouldn't be finished here and it wouldn't work because the directive wouldn't be rendered yet, so I call `setTimeout` in order to broadcast the event once the stack is empty and all transition code is done with.

Programatically navigating to the modal from another controller or service would look like this:

{% highlight javascript %}
state.go('usercp', { modal: true });
{% endhighlight %}

All in all this is a hack but it works perfectly, although I while writing this I realized that there's another "endorsed" way to achieve the same thing:

{% highlight javascript %}
modalRunBlock.$inject = ['$transitions', '$rootScope'];

function modalRunBlock(transitionService, $rootScope) {

  const shouldRunTransitionCallbackCriteria = {
    to: (state) => {
        return state.name === 'usercp';
      }
  };

  function showModal(transition){
      if(transition.params().modal){
        setTimeout(() => { this.$broadcast('toggleModal') }, 0);
      }
  }

  // Register the "showModal" hook with the TransitionService
  transitionService.onSuccess(shouldRunTransitionCallbackCriteria, showModal, { priority: 10, bind: $rootScope });
}

app.run(modalRunBlock);
{% endhighlight %}


I think this would be "cannon" way of achieving this mainly because it at least is hooking to the last transition hook that is activated, eventhough we still need the `setTimeout` because the `exampleModal` directive and the `scope.$on('toggleModal')` event listener have not loaded yet.

All in all I like my solution and will keep it as is, hope this gets indexed by search engines and it helps someone.
